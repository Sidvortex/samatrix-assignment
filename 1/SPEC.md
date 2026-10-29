# SPEC — Leave Requests for Comsky HRMS

> Written before any code. Any change to this spec after implementation starts goes in §12 Changelog, with the reason.

**Stack:** TypeScript end to end · Node + Express 5 · PostgreSQL 16 + Prisma · React + Vite · Zod · Vitest + Supertest · Docker (Postgres for dev and tests)

**Hosting:** Neon (Postgres) · Render (API) · Vercel (frontend)

---

## 0. Decisions at a glance

| # | Question | Decision |
|---|----------|----------|
| D1 | Who approves a manager's leave? | The company admin. Every non-admin user has a `managerId`: employees → a manager, managers → the admin. Approval right = "I am the requester's direct manager". The admin cannot apply for leave (README limitation). |
| D2 | Request spanning two calendar years | Rejected (`CROSSES_YEAR_BOUNDARY`). |
| D3 | Pending leave at submission | Counts against the balance: `workingDays ≤ allocated − used − pending`. Approval re-checks authoritatively against `allocated − used`. |
| D4 | Overlap enforcement | Postgres exclusion constraint (`btree_gist`) plus an app-level pre-check. SQLSTATE `23P01` on that constraint maps explicitly to `409 OVERLAPPING_REQUEST`. Detection lives in one function, `isOverlapViolation(err)`, chosen after a spike (§6.4). |
| D5 | Tenant isolation | `companyId` on every tenant table, composite FKs, `companyId` taken only from the authenticated user, tenant-scoped data layer, `404` across tenants. RLS is listed as a next step in the README. |
| D6 | Backdating | Only `SICK` may start before today. `CASUAL`/`EARNED` must start today or later, otherwise `422 BACKDATE_NOT_ALLOWED`. |
| D7 | What is "today"? | The calendar date in the **company's** timezone (`company.timezone`, default `Asia/Kolkata`). It is never the server's UTC date. |
| D8 | Validation | Zod on every endpoint (body, params and query). Unknown fields are rejected. Reason is required, 1–500 chars. At most 30 working days per request. |
| D9 | Test database | Real Postgres in Docker. No SQLite, no DB mocks. |
| D10 | Secrets | `JWT_SECRET` comes from env. The server refuses to start without it. Nothing is hardcoded. |
| D11 | Route list | One exported route registry (method, path, roles, schemas, handler) is the single source of truth. The app registers routes from it, and the role and tenant-isolation tests loop over it. No Express internals are introspected. |
| D12 | Hosting | Neon for Postgres (free tier doesn't expire), Render for the API, Vercel for the frontend. `btree_gist` on Neon is verified before anything depends on it. |

---

## 1. Roles, reporting lines and visibility

### 1.1 Roles

| Role | Can apply for leave | Approves | Sees |
|------|----|----|----|
| `EMPLOYEE` | yes | — | own requests and balances |
| `MANAGER` | yes | direct reports (the employees) | own requests and balances, plus direct reports' requests |
| `ADMIN` | **no** | direct reports (the managers) | every request and balance in their company; edits allocations |

A user has exactly one role. A manager is also an employee for their own leave, through the same `/me/*` endpoints.

### 1.2 Reporting-line invariants

These are enforced by the seed and by DB checks where possible. There is no API to change reporting lines.

- `ADMIN.managerId IS NULL` (DB check).
- `MANAGER.managerId` = the admin of the same company.
- `EMPLOYEE.managerId` = a manager of the same company.
- `managerId ≠ id` (DB check).
- `managerId` is in the same company. A composite FK makes a cross-company manager impossible.

### 1.3 Who can see a leave request

A request is **visible** to an actor if and only if all of the following hold:

- it is in the actor's company, **and**
- at least one of:
  - the actor owns it
  - the actor is the requester's direct manager
  - the actor is `ADMIN`

If a request is not visible to you, it does not exist for you, so the response is `404`. If it is visible but the action is not allowed, the response is `403`.

### 1.4 Who can act on a request

| Action | Allowed when |
|---|---|
| cancel | actor is the owner **and** status is `PENDING` |
| approve / reject | actor ≠ owner **and** actor is the requester's direct manager **and** status is `PENDING` |

---

## 2. Data model

Tables are snake_case via Prisma `@@map`/`@map`. Fields are camelCase in TypeScript. All IDs are `uuid` (v4, generated in the DB with `gen_random_uuid()`).

### 2.1 Enums

```
role          : EMPLOYEE | MANAGER | ADMIN
leave_type    : CASUAL | SICK | EARNED
leave_status  : PENDING | APPROVED | REJECTED | CANCELLED
```

### 2.2 `company`

| column | type | notes |
|---|---|---|
| id | uuid PK | |
| name | text NOT NULL | |
| timezone | text NOT NULL DEFAULT `'Asia/Kolkata'` | IANA name. Validated in the seed by `Intl.DateTimeFormat`. Not editable via the API. |
| created_at | timestamptz NOT NULL DEFAULT now() | |

### 2.3 `user`

| column | type | notes |
|---|---|---|
| id | uuid PK | |
| company_id | uuid NOT NULL → company(id) | |
| email | text NOT NULL | stored lowercase |
| name | text NOT NULL | |
| role | role NOT NULL | |
| manager_id | uuid NULL | |
| password_hash | text NOT NULL | bcrypt |
| created_at | timestamptz NOT NULL DEFAULT now() | |

Constraints and indexes:

- `UNIQUE (email)`. Global, because login takes only an email.
- `UNIQUE (company_id, id)`. This is the target for composite FKs.
- `FOREIGN KEY (company_id, manager_id) REFERENCES user(company_id, id) ON DELETE RESTRICT`. Under the default MATCH SIMPLE, a NULL `manager_id` skips the check, which is the behaviour we want.
- `CHECK (manager_id IS NULL OR manager_id <> id)`
- `CHECK ((role = 'ADMIN') = (manager_id IS NULL))`
- `INDEX (company_id, manager_id)`. Used by "my direct reports".

### 2.4 `leave_balance`

There is one row per user per year per leave type.

| column | type | notes |
|---|---|---|
| id | uuid PK | |
| company_id | uuid NOT NULL | |
| user_id | uuid NOT NULL | |
| year | int NOT NULL | calendar year |
| leave_type | leave_type NOT NULL | |
| allocated_days | int NOT NULL | set by the admin |
| used_days | int NOT NULL DEFAULT 0 | changed **only** by approval |
| updated_at | timestamptz NOT NULL | |

Constraints and indexes:

- `UNIQUE (company_id, user_id, year, leave_type)`. This is the natural key, used for upserts and the approval update.
- `FOREIGN KEY (company_id, user_id) REFERENCES user(company_id, id) ON DELETE RESTRICT`
- `CHECK (allocated_days >= 0 AND used_days >= 0)`
- `CHECK (used_days <= allocated_days)`. This is the backstop for rules 4 and 5. Even a buggy code path cannot overdraw.
- `CHECK (year BETWEEN 2000 AND 2100)`

Days are integers because there are no half-days. "Pending" is **not stored**. It is computed as `SUM(working_days)` over `PENDING` requests for that user, type and year. Storing it would add a second counter to keep consistent, for no benefit at this scale.

**Ledger invariant.** `used_days` is a stored counter, so it must always equal the sum of `working_days` over the user's `APPROVED` requests for the same type and year (year of `start_date`). Postgres can't enforce this cross-table rule cheaply, so `checkInvariants()` (§9) verifies it. Every writer of `APPROVED` must keep it true: the approve transaction (§7.1) **and the seed** (§11.1).

### 2.5 `leave_request`

| column | type | notes |
|---|---|---|
| id | uuid PK | |
| company_id | uuid NOT NULL | |
| employee_id | uuid NOT NULL | the requester (an employee or a manager) |
| leave_type | leave_type NOT NULL | |
| start_date | date NOT NULL | |
| end_date | date NOT NULL | inclusive |
| working_days | int NOT NULL | computed server-side at submission; Mon–Fri in `[start, end]` |
| reason | text NOT NULL | 1–500 chars |
| status | leave_status NOT NULL DEFAULT 'PENDING' | |
| decided_by_id | uuid NULL | approver or rejecter |
| decided_at | timestamptz NULL | |
| decision_comment | text NULL | ≤ 500 chars |
| cancelled_at | timestamptz NULL | |
| created_at | timestamptz NOT NULL DEFAULT now() | |
| updated_at | timestamptz NOT NULL | |

Constraints and indexes:

- `UNIQUE (company_id, id)`
- `FOREIGN KEY (company_id, employee_id) REFERENCES user(company_id, id) ON DELETE RESTRICT`
- `FOREIGN KEY (company_id, decided_by_id) REFERENCES user(company_id, id) ON DELETE RESTRICT`
- `CHECK (end_date >= start_date)`
- `CHECK (date_part('year', start_date) = date_part('year', end_date))`. Enforces D2 at the DB level too.
- `CHECK (working_days BETWEEN 1 AND 30)`
- `CHECK (char_length(reason) BETWEEN 1 AND 500)`
- `CHECK ((status IN ('APPROVED','REJECTED')) = (decided_by_id IS NOT NULL AND decided_at IS NOT NULL))`
- `CHECK ((status = 'CANCELLED') = (cancelled_at IS NOT NULL))`
- `CHECK (decided_by_id IS NULL OR decided_by_id <> employee_id)`. Rule 6 backstop: no self-approval even through a bug.
- **Exclusion constraint (rule 3):**
  ```sql
  CREATE EXTENSION IF NOT EXISTS btree_gist;
  ALTER TABLE leave_request ADD CONSTRAINT leave_request_no_overlap
    EXCLUDE USING gist (
      employee_id WITH =,
      daterange(start_date, end_date, '[]') WITH &&
    ) WHERE (status IN ('PENDING', 'APPROVED'));
  ```
  This applies across leave types. Rejected and cancelled rows drop out of the predicate, so they never block a new request. The constraint's GiST index also serves the app-level overlap pre-check.
- `INDEX (company_id, employee_id, status)`. Used by "my requests" and the pending sum.
- `INDEX (company_id, status, created_at DESC)`. Used by the admin "all requests" list and approvals.

### 2.6 Prisma notes and risks

- The exclusion constraint, the `btree_gist` extension and the cross-column CHECKs are not expressible in Prisma. They go in a **hand-edited migration** (`prisma migrate dev --create-only`, then append SQL). The README calls this out so nobody runs `db push`, which would skip them.
- Composite FKs are declared as Prisma relations (`fields: [companyId, employeeId], references: [companyId, id]`). If Prisma rejects several relations sharing `companyId` on one model, those FKs move into the raw-SQL migration and Prisma keeps single-column relations for typing. Either way the DB enforces them.
- `@db.Date` columns come back from Prisma as JS `Date` at UTC midnight. **Rule:** the API only ever exchanges dates as `"YYYY-MM-DD"` strings. Conversion happens in one helper that uses UTC getters only. Local-time `Date` methods are banned in date code.

---

## 3. Dates, "today" and working days

- **Calendar dates are timezone-free.** `start_date`/`end_date` are plain dates. Weekends are Saturday and Sunday for every tenant, so counting working days needs no timezone.
- **"Today"** = `formatDate(clock.now(), company.timezone)`, implemented with `Intl.DateTimeFormat('en-CA', { timeZone })` → `YYYY-MM-DD`.
  - Example: at `2026-11-02T19:00Z` it is `00:30` on Nov 3 in Asia/Kolkata. "Today" is `2026-11-03`, while the server's UTC date is still Nov 2.
- **Clock injection:** `createApp({ clock })`. Production uses `{ now: () => new Date() }`; tests pass a fixed clock. No code calls `new Date()` for "now" except the default clock.
- **Working days** = count of Mon–Fri dates in `[start, end]` inclusive. Pure function, unit-tested.
- **Balance year** = the year of `start_date`. D2 guarantees this equals the year of `end_date`.

---

## 4. Authentication

- `POST /api/auth/login { email, password }`
  - Email is lowercased, then the user is looked up.
  - Success returns `{ token, user }`.
  - Unknown email and wrong password both return the **byte-identical** `401 INVALID_CREDENTIALS`.
- **Constant-work login (no email enumeration by timing):**
  - At startup the server computes one `DUMMY_HASH` with `bcrypt.hash(<random bytes>, BCRYPT_COST)`.
  - Every login does **exactly one** `bcrypt.compare`. That is against the user's hash when the user exists, otherwise against `DUMMY_HASH`. There is no early return before the compare.
  - `BCRYPT_COST` (default 10) is the single cost used for the seed's hashes **and** the dummy hash. bcrypt's run time depends on the cost embedded in the hash, so a mismatch would reintroduce the timing gap.
- **JWT:**
  - HS256, pinned on verify. `alg:none` and other algorithms are rejected.
  - Secret from `JWT_SECRET`. The server exits at startup if it is missing or shorter than 32 chars.
  - Expiry 8h.
  - Payload is `{ sub: userId }` only.
- **Auth middleware:**
  - Reads `Authorization: Bearer <token>`.
  - Verifies the token, then loads `{ id, companyId, role, managerId }` from the DB **on every request**. Role and company are never trusted from the token or the client.
  - A missing, invalid or expired token, or a user that no longer exists, returns `401 UNAUTHENTICATED`.
- **User switcher:** the frontend ships a static list of the seed users' emails (demo only) and calls `/auth/login`. There is no API endpoint that lists users across companies. The seed password comes from `SEED_PASSWORD` and is documented in the README for demo use.

---

## 5. Tenant isolation (rule 7)

Isolation is enforced in four layers. Any one of them alone should be enough to stop a cross-tenant read or write.

1. **Identity is the only source of `companyId`.** It comes from the DB-loaded user in the auth middleware. No endpoint accepts `companyId` in the body, query, params or headers, and Zod `.strict()` rejects it as an unknown key.
2. **Tenant-scoped data layer.**
   - Route handlers never import the Prisma client.
   - All DB access goes through repository functions that take a `ctx: { userId, companyId, role }` and include `companyId` in **every** `where`, including the raw-SQL statements in §7.
   - Lookups by ID are `findFirst({ where: { id, companyId } })`, never `findUnique({ where: { id } })`.
3. **Composite foreign keys** `(company_id, x_id) → user(company_id, id)`. A row in company A physically cannot reference a user in company B, so a bug in the app layer cannot plant cross-tenant links.
4. **`404` for everything out of scope.** Another company's ID and a nonexistent ID return identical responses (same status, code and message), so IDs cannot be probed for existence. UUIDv4 IDs are hard to guess, but the design does not rely on that.
   - The 404 must come from **the same code path** in both cases. A scoped query (`WHERE id = $1 AND company_id = $2`) returns nothing either way.
   - Never load by ID and then compare `companyId` in code. That adds a branch whose timing and logging differ between "exists elsewhere" and "doesn't exist".

**Next step (README, not built):** Postgres row-level security keyed on a per-transaction `app.company_id` setting, with the app connecting as a non-owner role. This is defence in depth if layer 2 is ever bypassed.

---

## 6. API contract

**Conventions**
- Base path `/api`. JSON in and out. `express.json({ limit: '10kb' })`.
- Dates are `"YYYY-MM-DD"`. Timestamps are ISO-8601 UTC.
- Success responses return the resource or `{ items: [...] }`. There is no envelope.
- Error shape:
  ```json
  { "error": { "code": "OVERLAPPING_REQUEST", "message": "human readable", "details": [] } }
  ```
- **Zod everywhere.** Every route validates `params`, `query` and `body` with `.strict()` schemas. Unknown keys → `400 VALIDATION_ERROR`. `details` is a list of `{ path, code, message }`.
- **Check order on every route:**
  1. auth (`401`)
  2. role gate (`403 FORBIDDEN_ROLE`)
  3. Zod (`400`)
  4. tenant-scoped load and visibility (`404`)
  5. action permission (`403`)
  6. state and business rules (`409`/`422`)

### 6.0 Route registry

`server/src/routes.ts` exports `routes: RouteDef[]`. It is the only place a route is defined.

```ts
type RouteDef = {
  method: 'get' | 'post' | 'put';
  path: string;                       // Express path, e.g. '/api/requests/:id/approve'
  roles: Role[] | 'public';
  schemas: { params?: ZodType; query?: ZodType; body?: ZodType };  // all .strict()
  handler: (req: ValidatedRequest, ctx: Ctx) => Promise<Response>;
};
```

`createApp()` loops over `routes` and registers each one as follows:

```
app[method](path, authenticate?, requireRole(roles), validate(schemas), handler)
```

Then it adds the 404 fallthrough and the error middleware. Nothing else calls `app.get/post/…` or creates an `express.Router`. This rule is checked in review and stated in the README.

The tests import the same array:
- **Role gate:** for every route and every role *not* in `roles` → `403 FORBIDDEN_ROLE`.
- **Tenant isolation (§9, rule 7):** for every route whose path contains a `:param`, the test looks up a cross-tenant probe keyed by `` `${method} ${path}` ``. A route with no probe **fails the test**, so adding an ID route forces adding its isolation check.

### 6.1 DTOs

```ts
type UserRef = { id: string; name: string };

type LeaveRequestDTO = {
  id: string;
  employee: UserRef;
  leaveType: 'CASUAL' | 'SICK' | 'EARNED';
  startDate: string;          // YYYY-MM-DD
  endDate: string;
  workingDays: number;
  reason: string;
  status: 'PENDING' | 'APPROVED' | 'REJECTED' | 'CANCELLED';
  createdAt: string;
  decision: null | { by: UserRef; at: string; comment: string | null };
  cancelledAt: string | null;
};

type BalanceDTO = {
  leaveType: 'CASUAL' | 'SICK' | 'EARNED';
  year: number;
  allocatedDays: number;
  usedDays: number;
  pendingDays: number;        // computed
  availableDays: number;      // allocated − used − pending
};

type MeDTO = {
  user: { id: string; name: string; email: string; role: 'EMPLOYEE' | 'MANAGER' | 'ADMIN'; manager: UserRef | null };
  company: { id: string; name: string; timezone: string; today: string };
};
```

### 6.2 Endpoints

| Method | Path | Roles | Purpose |
|---|---|---|---|
| GET | `/api/health` | public | Liveness + DB reachability (used by deploy checks) |
| POST | `/api/auth/login` | public | Log in |
| GET | `/api/me` | any | Current user, company, company "today" |
| GET | `/api/me/balances?year=` | EMPLOYEE, MANAGER | Own balances (all 3 types) |
| GET | `/api/me/requests?status=&year=` | EMPLOYEE, MANAGER | Own requests |
| POST | `/api/me/requests` | EMPLOYEE, MANAGER | Apply for leave |
| GET | `/api/requests/:id` | any | One request, if visible |
| POST | `/api/requests/:id/cancel` | EMPLOYEE, MANAGER | Owner cancels a pending request |
| GET | `/api/approvals?status=` | MANAGER, ADMIN | Direct reports' requests (default `PENDING`) |
| POST | `/api/requests/:id/approve` | MANAGER, ADMIN | Approve (atomic deduction) |
| POST | `/api/requests/:id/reject` | MANAGER, ADMIN | Reject with comment |
| GET | `/api/admin/requests?status=&year=` | ADMIN | Every request in the company |
| GET | `/api/admin/balances?year=` | ADMIN | Balances table for the company |
| PUT | `/api/admin/balances/:userId/:year/:leaveType` | ADMIN | Set (upsert) an allocation |

Shared query params:
- `year` is an int from 2000 to 2100. It defaults to the current year in the company's timezone.
- `status` is a `leave_status` value.
- Lists are ordered `created_at DESC` and capped at 200 rows. There is no pagination; see §10.

#### GET `/api/health`
- Runs `SELECT 1`.
- `200 { status: 'ok', db: 'ok' }` or `503 { status: 'degraded', db: 'down' }`.
- Exposes nothing else: no version, env, extension list or error text.

#### POST `/api/auth/login`
- Body: `{ email: string (email, ≤254), password: string (1–200) }`
- `200 { token, user: MeDTO['user'] }`
- Errors: `400 VALIDATION_ERROR`, `401 INVALID_CREDENTIALS`
- Constant-work rule: see §4.

#### GET `/api/me`
- `200 MeDTO`

#### GET `/api/me/balances?year=`
- `200 { items: BalanceDTO[] }`. A type with no balance row for that year is omitted.
- Errors: `403 FORBIDDEN_ROLE` (admin)

#### GET `/api/me/requests?status=&year=`
- `200 { items: LeaveRequestDTO[] }`

#### POST `/api/me/requests`

Body (`.strict()`):
```ts
{
  leaveType: 'CASUAL' | 'SICK' | 'EARNED',
  startDate: 'YYYY-MM-DD',   // regex + real calendar date (2026-02-30 rejected)
  endDate:   'YYYY-MM-DD',
  reason: string             // trimmed, 1–500 chars
}
```

Zod refinements on pure input. These come back as `400 VALIDATION_ERROR` with the rule code in `details[].code`:
- `END_BEFORE_START`
- `CROSSES_YEAR_BOUNDARY`
- `NO_WORKING_DAYS`: the range is weekend-only
- `TOO_MANY_WORKING_DAYS`: more than 30

Server checks, in order:
1. `leaveType ≠ SICK` and `startDate < today(company.timezone)` → `422 BACKDATE_NOT_ALLOWED`
2. No balance row for `(user, year(startDate), leaveType)` → `409 NO_BALANCE_FOR_YEAR`
3. `workingDays > allocated − used − pending` → `409 INSUFFICIENT_BALANCE`. This is a soft check, not locked; see §7.3.
4. App-level overlap pre-check against own `PENDING`/`APPROVED` requests → `409 OVERLAPPING_REQUEST`
5. Insert. A `23P01` on `leave_request_no_overlap` → `409 OVERLAPPING_REQUEST`

Response: `201 LeaveRequestDTO` (status `PENDING`).

#### GET `/api/requests/:id`
- `200 LeaveRequestDTO` if visible (§1.3), otherwise `404 NOT_FOUND`

#### POST `/api/requests/:id/cancel`
- Body: `{}` (`.strict()`)
- Errors:
  - `404` if not visible
  - `403 NOT_REQUEST_OWNER` if visible but not the actor's own (e.g. the actor's manager)
  - `409 REQUEST_NOT_PENDING`
- `200 LeaveRequestDTO` (status `CANCELLED`). The balance is untouched, because pending never deducts.

#### GET `/api/approvals?status=`
- `200 { items: LeaveRequestDTO[] }`: requests whose employee's `managerId = actor.id`
- A manager sees their employees' requests. The admin sees the managers' requests. Nobody sees their own here.

#### POST `/api/requests/:id/approve`
- Body: `{ comment?: string (trimmed, ≤500) }`
- Checks, in order:
  1. Not visible → `404 NOT_FOUND`
  2. `employeeId = actor.id` → `403 SELF_APPROVAL_FORBIDDEN`
  3. Actor is not the requester's direct manager → `403 NOT_DIRECT_MANAGER`. Reachable only by the admin, who can see everything. A manager looking at a non-report hits step 1.
  4. The atomic transaction in §7.1 runs. Its failures:
     - `409 REQUEST_NOT_PENDING`
     - `409 NO_BALANCE_FOR_YEAR`
     - `409 INSUFFICIENT_BALANCE`
- `200 LeaveRequestDTO` (status `APPROVED`). If the deduction fails, the request **stays `PENDING`**, so the manager can still reject it.

#### POST `/api/requests/:id/reject`
- Body: `{ comment: string (trimmed, 1–500) }`. Required.
- Checks 1–3 are the same as approve. Then a guarded transition (§7.2) → `409 REQUEST_NOT_PENDING`.
- `200 LeaveRequestDTO` (status `REJECTED`)

#### GET `/api/admin/requests?status=&year=`
- `200 { items: LeaveRequestDTO[] }`, company-wide

#### GET `/api/admin/balances?year=`

```ts
200 {
  year: number,
  items: Array<{
    user: { id; name; email; role: 'EMPLOYEE' | 'MANAGER'; manager: UserRef | null },
    balances: { CASUAL: BalanceDTO | null; SICK: BalanceDTO | null; EARNED: BalanceDTO | null }
  }>
}
```

Admins are excluded from the rows because they hold no balances.

#### PUT `/api/admin/balances/:userId/:year/:leaveType`
- Params:
  - `userId`: uuid
  - `year`: int 2000–2100
  - `leaveType`: enum
- Body: `{ allocatedDays: int 0–365 }`
- Errors:
  - `404 NOT_FOUND`: user not in the actor's company
  - `422 USER_HAS_NO_BALANCES`: target is an admin
  - `409 ALLOCATION_BELOW_USED`
- Upsert (single statement, §7.4). `200 BalanceDTO`

### 6.3 Error codes

| HTTP | code | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Zod failure. `details[].code` carries `END_BEFORE_START`, `CROSSES_YEAR_BOUNDARY`, `NO_WORKING_DAYS`, `TOO_MANY_WORKING_DAYS`, `UNRECOGNIZED_KEYS` or a Zod issue code |
| 401 | `INVALID_CREDENTIALS` | bad login |
| 401 | `UNAUTHENTICATED` | missing, invalid or expired token; user gone |
| 403 | `FORBIDDEN_ROLE` | role not allowed on route |
| 403 | `SELF_APPROVAL_FORBIDDEN` | approve or reject own request |
| 403 | `NOT_DIRECT_MANAGER` | visible request, but actor isn't the requester's manager |
| 403 | `NOT_REQUEST_OWNER` | cancel someone else's visible request |
| 404 | `NOT_FOUND` | unknown, or not visible (incl. other company), or unknown route |
| 409 | `OVERLAPPING_REQUEST` | app pre-check **or** SQLSTATE `23P01` on `leave_request_no_overlap` |
| 409 | `INSUFFICIENT_BALANCE` | at submission (incl. pending) or at approval (excl. pending). Also SQLSTATE `23514` on the `used_days <= allocated_days` check, as a backstop |
| 409 | `REQUEST_NOT_PENDING` | cancel, approve or reject a request that isn't `PENDING` (incl. losing a race) |
| 409 | `NO_BALANCE_FOR_YEAR` | no balance row for user, year and type |
| 409 | `ALLOCATION_BELOW_USED` | admin sets allocation below `used_days` |
| 422 | `BACKDATE_NOT_ALLOWED` | `CASUAL`/`EARNED` starting before company-today |
| 422 | `USER_HAS_NO_BALANCES` | admin tries to set an admin's balance |
| 500 | `INTERNAL_ERROR` | anything else. Logged server-side; no stack or SQL in the body |

### 6.4 DB error detection

All knowledge of Prisma and Postgres error shapes lives in **one module**, `server/src/db/pgErrors.ts`:

| function | true when |
|---|---|
| `isOverlapViolation(err)` | SQLSTATE `23P01` **and** constraint `leave_request_no_overlap` |
| `isBalanceCheckViolation(err)` | SQLSTATE `23514` **and** the `used_days <= allocated_days` check constraint |

The error middleware calls these and maps them to `409 OVERLAPPING_REQUEST` and `409 INSUFFICIENT_BALANCE`. Any other DB error → `500`. No other file inspects `err.code`, `err.meta` or `err.message`.

**Spike, before the request service is written** (PLAN step 8):
1. Against the dev DB, insert an overlapping request through Prisma `create()`.
2. Log the full error: class name, `code`, `meta`, `message` and `cause`.
3. Do the same through `$queryRaw` for comparison.
4. Record the observed shapes in §12 Changelog and **pick one insert path**:
   - If `create()` exposes SQLSTATE and constraint name in structured fields → use `create()`.
   - Otherwise → the request insert uses parameterized `$queryRaw`, and `isOverlapViolation` reads its structured fields.
   - Message-string matching is a last resort, and only inside `isOverlapViolation`.
5. Apply the same spike and decision to the balance CHECK violation.

**Own tests (real DB):**
- `isOverlapViolation` returns **true** for a real overlap violation produced through the chosen insert path.
- It returns **false** for:
  - a unique violation
  - an FK violation
  - the balance CHECK violation
  - another CHECK violation
  - a plain `Error`
  - `undefined`
- `isBalanceCheckViolation` gets the mirror-image set of tests.

---

## 7. Concurrency and atomicity (rules 4 and 5)

The default isolation is **READ COMMITTED**. Correctness comes from **guarded single-row `UPDATE`s inside one transaction**, not from read-then-write.

Under READ COMMITTED, Postgres works like this:
- A second `UPDATE` on a row locked by an in-flight transaction **blocks**.
- When the first transaction commits, the second **re-evaluates its `WHERE` against the committed row** (EvalPlanQual).
- So a guard like `status = 'PENDING'` or `used_days + d <= allocated_days` is always checked against the latest committed value.

### 7.1 Approve

All of this runs in one `prisma.$transaction(async tx => …)`.

```sql
-- step 1: claim the request (row lock on leave_request)
UPDATE leave_request
   SET status = 'APPROVED', decided_by_id = $actor, decided_at = now(),
       decision_comment = $comment, updated_at = now()
 WHERE id = $id AND company_id = $companyId AND status = 'PENDING'
RETURNING employee_id, leave_type, start_date, working_days;
-- 0 rows → ROLLBACK, 409 REQUEST_NOT_PENDING

-- step 2: deduct (row lock on leave_balance), guarded
UPDATE leave_balance
   SET used_days = used_days + $days, updated_at = now()
 WHERE company_id = $companyId AND user_id = $employeeId
   AND year = $year AND leave_type = $type
   AND used_days + $days <= allocated_days;
-- 0 rows → read the row to tell missing (409 NO_BALANCE_FOR_YEAR)
--          from short (409 INSUFFICIENT_BALANCE), then ROLLBACK
COMMIT;
```

Step 2 is raw SQL (`tx.$executeRaw` with tagged template parameters) because Prisma's `updateMany` can't express `used + d <= allocated`.

**Why two approvals can't double-deduct:**

| Race | What happens |
|---|---|
| Same request approved twice concurrently (double-click, two tabs) | Both hit step 1 on the same row. One blocks until the other commits, then re-evaluates `status = 'PENDING'` → 0 rows → `REQUEST_NOT_PENDING`. Only the winner ever reaches step 2. |
| Two *different* pending requests for the same employee and type, whose sum exceeds the balance | Each wins its own step 1. Both reach step 2 on the **same balance row**, so one blocks. After the first commits, the second re-evaluates `used_days + d <= allocated_days` against the new `used_days` → 0 rows → `INSUFFICIENT_BALANCE`, and its step 1 is rolled back too. |
| Approve vs cancel, or approve vs reject, on one request | Same row lock in step 1. Whoever commits first wins; the other gets `REQUEST_NOT_PENDING`. |
| Approve vs admin allocation edit | Serialized on the balance row. Each guard re-evaluates against committed state. |

- **Backstop:** `CHECK (used_days <= allocated_days)`. If a future code path forgets the guard, the DB still refuses.
- **Deadlocks:** approval always locks `leave_request` then `leave_balance`. Cancel and reject lock only `leave_request`. The allocation edit locks only `leave_balance`. There is no lock-order cycle.

### 7.2 Reject and cancel

A single guarded `UPDATE … WHERE id AND company_id AND status = 'PENDING'`, plus `employee_id = $actor` for cancel. 0 rows → `REQUEST_NOT_PENDING`. There are no balance writes.

### 7.3 Submission

The balance check at submission is a **soft, unlocked check** (D3):
- Two concurrent submissions can both pass it and over-book *pending* days.
- That is accepted, because nothing is deducted until approval, and approval re-checks under lock (rule 4).
- The **overlap** rule, by contrast, is hard-enforced at submission by the exclusion constraint, so concurrent identical submissions produce exactly one row.

### 7.4 Admin allocation edit

One statement:

```sql
INSERT INTO leave_balance (id, company_id, user_id, year, leave_type, allocated_days, used_days, updated_at)
VALUES (gen_random_uuid(), $companyId, $userId, $year, $type, $alloc, 0, now())
ON CONFLICT (company_id, user_id, year, leave_type)
DO UPDATE SET allocated_days = EXCLUDED.allocated_days, updated_at = now()
   WHERE leave_balance.used_days <= EXCLUDED.allocated_days
RETURNING *;
-- no row returned → 409 ALLOCATION_BELOW_USED
```

Lowering an allocation below `used + pending` is allowed. The pending requests then fail at approval with `INSUFFICIENT_BALANCE`, which is exactly rule 4.

---

## 8. Edge cases (decided)

**Dates and working days**
- A range that starts or ends on a weekend is fine; weekend days just don't count. A weekend-only range → `NO_WORKING_DAYS`.
- A single-day weekday request (`start = end`) counts as 1 working day.
- Exactly 30 working days is accepted; 31 → `TOO_MANY_WORKING_DAYS`.
- Invalid calendar dates (`2026-02-30`), wrong formats or timestamps → `400`.

**"Today" and backdating**
- Between 00:00 and 05:30 IST, a casual leave starting "today" (IST) is accepted, even though UTC is still on yesterday.
- A sick leave backdated into the previous year works if that year's balance row exists, and must not itself cross a year boundary.
- A request in a future year with no balance row yet → `NO_BALANCE_FOR_YEAR`.

**Overlaps**
- Touching on the same day overlaps (inclusive ranges) → `409`. Adjacent ranges (Fri / next Mon) don't.
- An overlap with a different leave type is still an overlap.
- An overlap with a `REJECTED`/`CANCELLED` request is allowed.

**Balances**
- Approving after the admin lowered the allocation → `INSUFFICIENT_BALANCE`, and the request stays `PENDING`.
- Rejecting never checks the balance.
- Approving a request whose dates are already in the past is allowed.
- Cancelling a pending request never touches `used_days`.
- Setting an allocation to 0 with 0 used is allowed.

**Approval races**
- A double-click on approve → the second gets `REQUEST_NOT_PENDING`.
- A manager's own request shows in their My leave and in the admin's Approvals. It never appears in their own Approvals, and approving it by ID → `SELF_APPROVAL_FORBIDDEN`.

**Auth and input**
- A token for a user who no longer exists, an expired token, a token signed with a different secret, or one using `alg: none` → `401`.
- Login email is case-insensitive (lowercased).
- Unknown body or query fields (e.g. `companyId`, `status`, `workingDays` on create) → `400 UNRECOGNIZED_KEYS`. The client can't set computed or server-owned fields.
- A malformed UUID in a path param → `400`. A well-formed UUID that isn't visible → `404`.

---

## 9. Tests

**Infrastructure**
- `docker compose` service `db-test` runs `postgres:16` on port 5433, with tmpfs storage for speed.
- `npm test`:
  1. `prisma migrate deploy` against `DATABASE_URL_TEST`. This applies the raw-SQL constraints; `db push` is never used.
  2. Vitest.
- Test files run sequentially against one DB. Each test file truncates all tables and reseeds a deterministic fixture.
- **Reset guard:**
  - Every truncate (the test `resetDb()` and the `db:reset` script) first calls `assertSafeToReset(url)`.
  - It throws unless the host is `localhost`, `127.0.0.1`, `::1`, `db` or `db-test`.
  - So a mistyped `DATABASE_URL_TEST` pointing at Neon fails loudly instead of wiping the live demo.
- **`checkInvariants(db)`** (`src/db/invariants.ts`, read-only) checks:
  - The ledger invariant (§2.4) holds for every balance row.
  - Every `APPROVED` request has a matching balance row.
  - Every decided request's `decided_by_id` is the requester's `manager_id`.

  It runs:
  - at the end of the seed (which aborts and rolls back on failure)
  - after every approval, concurrency and allocation test
  - as `npm run check:invariants`, which is safe against Neon because it's read-only
- The Prisma pool size is ≥ 5 (`connection_limit`), so concurrent requests really do use separate connections.
- No SQLite and no DB mocks. The exclusion constraint, the row locks and SQLSTATE mapping are what's being tested.
- HTTP tests use Supertest against `createApp({ clock })` with a fixed clock.
- Concurrency tests fire requests with `Promise.all` and repeat for **10 rounds** each, to catch flaky interleavings.

**Coverage by rule**

| Rule | Tests |
|---|---|
| **3** overlap | <ul><li>Overlap with a PENDING request → 409.</li><li>Overlap with an APPROVED request → 409.</li><li>Overlap with REJECTED or CANCELLED → 201.</li><li>A different leave type still conflicts.</li><li>Same-day touch → 409; adjacent → 201.</li><li>**Two identical submissions fired concurrently → exactly one 201 and one 409 `OVERLAPPING_REQUEST`, and exactly one row in the DB.**</li><li>A direct DB insert bypassing the app → `23P01`. This proves the constraint exists, not only the pre-check.</li></ul> |
| **4** balance at approval | Submit within balance → admin lowers allocation → approve → 409 `INSUFFICIENT_BALANCE`, request still PENDING, `used_days` unchanged. Plus a submission-time rejection, including the pending-days case. |
| **5** atomic approval | <ul><li>**(a) The plain rule-5 test, kept as its own test:** one pending request is approved twice concurrently by its manager (two tabs or a double-click). Exactly one 200 and one 409 `REQUEST_NOT_PENDING`. `used_days` increases by exactly the request's working days, once. The request has a single `decided_at`.</li><li>(b) Two pending requests whose sum exceeds the balance, approved concurrently. Setup: submit both, then the admin lowers the allocation, because the D3 submission check otherwise prevents this state. Result: one 200 and one 409 `INSUFFICIENT_BALANCE`; `used_days ≤ allocated_days`; the loser stays PENDING.</li><li>(c) Approve vs cancel concurrently → exactly one wins and the final state is consistent.</li></ul> |
| **6** manager limits | <ul><li>A manager approving or rejecting their own request → 403 `SELF_APPROVAL_FORBIDDEN`.</li><li>A manager acting on the other manager's report → 404.</li><li>The admin acting on an employee's request → 403 `NOT_DIRECT_MANAGER`.</li><li>The admin approving a manager's request → 200.</li><li>An employee calling approve → 403 `FORBIDDEN_ROLE`.</li><li>The DB check rejects `decided_by_id = employee_id` on a direct write.</li></ul> |
| **7** tenant isolation | <ul><li>**A loop over the shared route registry (§6.0).** Every route whose path has a `:param` must have a probe `{ actor role (one allowed on that route), params built from company B's fixture, valid body }`. A missing probe fails the test.</li><li>Each probe sends a real ID from company B as a company-A actor. The response must be `404 NOT_FOUND` and byte-identical to the response for a random nonexistent UUID. For mutating routes, the B row is re-read and must be unchanged.</li><li>Today the probed routes are `/requests/:id`, `/requests/:id/cancel`, `/requests/:id/approve`, `/requests/:id/reject` and `/admin/balances/:userId/:year/:leaveType`.</li><li>Every list endpoint, called as each role in A, returns zero rows belonging to B.</li><li>The body/query field `companyId` → 400.</li><li>A composite-FK test: a direct DB insert of a request in A whose `employee_id` is in B fails.</li></ul> |

**Also covered**
- **Login enumeration (§4):**
  - *(Deterministic)* `bcrypt.compare` is spied on. For an unknown email and for a wrong password it is called exactly once, and the two response bodies are byte-identical.
  - *(Coarse timing)* 20 interleaved attempts of each. The ratio of median latencies must fall between 0.5 and 2.0. The regression this guards against (skipping bcrypt) is around 100×, so these loose bounds won't flake.
- **Seed:** produces no invariant violations. A second run refuses and leaves the data untouched. `db:reset` refuses a non-local URL.
- `isOverlapViolation` and `isBalanceCheckViolation` (§6.4).
- The role gate, by looping over the route registry.
- Working-day counting (unit).
- "Today" across the IST midnight boundary: the fixed clock is set to `19:00Z`. A casual leave starting IST-today → 201; starting UTC-today (= IST-yesterday) → 422 `BACKDATE_NOT_ALLOWED`. Sick leave on the same date → 201.
- Per-company timezone: the fixture switches company B to `America/New_York` and asserts B's "today" differs from A's at the same instant.
- Zod `.strict()` rejects unknown fields on every body. Reason length 0 / 501 is rejected.
- `ALLOCATION_BELOW_USED`.
- Startup fails without `JWT_SECRET`.
- JWT `alg:none` is rejected.

---

## 10. Out of scope

| Item | Why / status |
|---|---|
| Signup, password reset, refresh tokens, login rate limiting | Brief says no signup; demo auth only |
| Admin applying for leave | **Known limitation (README)**: the admin has no approver |
| Editing a request | Cancel and re-apply instead |
| Cancelling an approved request (and refunding balance) | Brief allows cancel only while pending. README "next" |
| Public holidays, per-tenant weekend config, half-days | Weekend fixed to Sat/Sun |
| Accrual, carry-over, year rollover jobs | Admin sets each year's allocation manually |
| Changing reporting lines or roles via API | Seed-defined |
| Editing a company's timezone via API | Seed-defined |
| Notifications (email, in-app) | — |
| Full audit log | The request keeps `decided_by/at/comment` and `cancelled_at` only |
| Pagination, search, sorting options | Lists capped at 200, newest first |
| Delegated approval (manager on leave) | — |
| Postgres RLS | **README next step** (§5) |

---

## 11. Seed data, screens and run setup

### 11.1 Seed

There are 2 companies, both in `Asia/Kolkata`: **Acme Corp** and **Globex Ltd**.

Each company has:
- 1 admin
- 2 managers, each reporting to the admin
- 4 employees, 2 per manager
- Emails like `priya.manager@acme.test`
- Every user's password is set from `SEED_PASSWORD`

Balances:
- Created for every non-admin, for the current year in the company's timezone.
- Casual 12, Sick 10, Earned 15.

Sample requests:
- A few per company, covering PENDING, APPROVED, REJECTED and CANCELLED.
- One pending manager request, so the admin's Approvals page isn't empty.

**Seed consistency**
- The whole seed runs in one transaction.
- `working_days` is computed with the same helper the API uses.
- Each `APPROVED` sample increments its balance's `used_days` in that transaction. `decided_by_id` is the requester's direct manager, and `decided_at` is set.
- `REJECTED` samples get a comment. `CANCELLED` samples get `cancelled_at`.
- Samples never overlap per employee.
- Approved and pending totals stay within the allocation.
- Dates:
  - Decided and cancelled samples sit in the past within the current year.
  - Pending samples sit in the future within the current year. If too few weekdays remain in the year, pending samples fall back to backdated `SICK`.
- The seed ends with `checkInvariants()` and rolls back on any violation.

**Seed safety**
- The seed is **insert-only**. If `company` already has rows, it exits non-zero with "already seeded" and touches nothing.
- Wiping data is a separate script, `db:reset`, guarded by `assertSafeToReset` (local hosts only).
- To reset the live demo on Neon, do it deliberately in the Neon console (reset the branch), then seed. No script can do it.

### 11.2 Screens

Navigation is by role. The user switcher sits in the header. The UI is never the authority: it shows API error codes as messages.

- **My leave** (employee, manager)
  - Balance cards showing allocated, used, pending and available.
  - Apply form: type, dates, reason. It shows the computed working days as a hint; the server recomputes.
  - Request list with status badges, and Cancel on pending requests.
- **Approvals** (manager, admin)
  - Pending direct-report requests, each with Approve (optional comment) and Reject (required comment).
  - A 409 on approve shows the reason inline.
- **Admin balances**
  - Table of users × leave types for a selectable year.
  - Inline edit of allocated days.
- **Admin all requests**
  - Read-only, company-wide list with a status filter.

### 11.3 Run setup and hosting

**Repo and dev**
- **Repo layout:** `server/` (Express, Prisma, tests) and `web/` (Vite React).
- **Dev:** Vite proxies `/api` to the local server. Postgres runs in Docker.
- **Tests:** always run against the Docker `db-test`, never against Neon.

**Database: Neon**
- **Pre-flight:** before any schema work, run `CREATE EXTENSION IF NOT EXISTS btree_gist;` on Neon, then create and drop a throwaway table with an `EXCLUDE USING gist` constraint. Record the result in §12. If it fails, stop and revisit D4 before going further.
- **Connections:**
  - Runtime uses Neon's **pooled** connection string (`DATABASE_URL`).
  - Migrations use the **direct** string (`DIRECT_URL`), configured as Prisma's direct URL.
  - Interactive transactions run inside one pooled connection for their whole duration, so the §7 guarantees hold.
- **Migrations:** Render's build command runs `prisma migrate deploy` against `DIRECT_URL`, so schema and code ship together. Migrations in this project are additive only.
- **Seed:** run once, by hand, from a local machine with `DIRECT_URL`. It is never part of a deploy. It refuses if the database already has data (§11.1). Afterwards, run `npm run check:invariants` against Neon.

**Deploy early, deploy continuously**
- A bare skeleton goes live in PLAN step 6, before any business logic. It consists of `/api/health` on Render, backed by the migrated Neon schema, and a Vercel page calling it cross-origin.
- This proves the risky parts first: Neon pooled vs direct URLs, Prisma's engine on Render's build image, the hand-edited migration on Neon, CORS, and Vercel env vars.
- From then on, Render and Vercel auto-deploy from `main`, and later steps include a live smoke check.

**API: Render**
- Node web service.
- Build: install, `prisma generate`, `prisma migrate deploy`, `tsc`.
- Start: `node dist/server.js`.

**Frontend: Vercel**
- Static Vite build of `web/`.
- `VITE_API_URL` points at the Render URL.
- `vercel.json` rewrites all paths to `index.html` so client routes survive a refresh.

**CORS**
- The API allows exactly the origins in `CORS_ORIGIN` (comma-separated: the Vercel production URL; localhost in dev).
- The token travels in the `Authorization` header and there are no cookies, so CORS is non-credentialed.
- Vercel preview URLs are not allowed unless added.

**Env vars**
- Server: `DATABASE_URL`, `DIRECT_URL`, `DATABASE_URL_TEST`, `JWT_SECRET`, `SEED_PASSWORD`, `BCRYPT_COST`, `CORS_ORIGIN`, `PORT`
- Web: `VITE_API_URL`
- `.env.example` is committed; `.env` never is.

**Known hosting limitations (README)**
- Render's free tier spins the API down when idle, so the first request after a while takes on the order of a minute.
- Neon also suspends idle compute, which adds a short delay on wake.

---

## 12. Changelog

| Date | Change | Reason |
|---|---|---|
| 2026-10-29 | v1 | Initial spec from brief + clarifications D1–D10 |
| 2026-10-29 | v1.1 | 23P01 handling: removed "either way"; spike decides one insert path; detection isolated in `isOverlapViolation` with its own tests (§6.4). Rule-5 double-approval test made explicit as its own test. Express router introspection replaced by a shared route registry (§6.0, D11), because Express internals changed in v5. Hosting fixed to Neon/Render/Vercel with a btree_gist pre-flight (D12, §11.3). The prod frontend is now cross-origin, so a CORS allowlist was added. |
| 2026-10-29 | v1.2 | Review fixes. (1) The seed's APPROVED samples didn't update `used_days`: added the ledger invariant (§2.4), a seed consistency section, and `checkInvariants()`. (2) The seed's truncate-then-insert could wipe Neon if run twice: the seed is now insert-only and refuses a non-empty DB; wiping moved to a local-only `db:reset`; the test truncate is guarded too. (3) Login leaked which emails exist through timing (unknown email skipped bcrypt): one compare per login against a dummy hash of equal cost, with deterministic and coarse timing tests. Also added that cross-tenant 404s must share the code path of real misses. (4) Deploy-last was the biggest schedule risk: a walking skeleton now deploys at step 6, with auto-deploys after that, migrations in the Render build, and a `/api/health` endpoint. |
| 2026-10-29 | v1.3 | Scope cut line added to PLAN (hour-30 checkpoint). If behind, these are dropped in order: rule 5c approve-vs-cancel test, admin edit vs approval race test, America/New_York timezone test, admin all-requests page (the `GET /admin/requests` endpoint stays), GitHub Actions. All other §9 tests are must-have. Dropped items go to README limitations. |
| _pending_ | Neon btree_gist pre-flight result | PLAN step 2 |
| _pending_ | 23P01 / 23514 spike result + chosen insert path | PLAN step 8 |
