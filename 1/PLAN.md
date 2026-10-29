# PLAN — build order for SPEC.md

Every step is small, ends in **one commit**, and has a "done when" check.

**Order:** pre-flight → **walking skeleton deployed** → DB correctness → API → UI → final checks and docs.

Deployment happens at step 6, not at the end. From then on, every push to `main` auto-deploys, and production problems show up on day 1 instead of hour 44.

**Working rules for every step**
1. Give the agent the step and the relevant SPEC sections only. Don't hand it the whole plan.
2. Read the diff before committing. Check it against SPEC, especially:
   - the `companyId` in every `where`
   - the guarded `UPDATE`s
   - that no code path writes `APPROVED` without touching `used_days`
3. Note in `AGENT_LOG.md` what was accepted, what was corrected and why.
4. `npm test` is green before the commit.
5. If a step changes a decision, update SPEC §12 Changelog in the same commit.
6. After step 6, if a push breaks the live deploy, fixing it comes before the next step.

Rough budget: pre-flight 1h · skeleton and deploy 4h · DB 5h · API 15h · UI 8h · final 3h. That leaves about 12h of slack in the 48.

## Cut line

**Checkpoint at hour 30 of the 48h clock.** If step 21 (API review pass) isn't committed by then, I'm behind. In that case, drop these items **in this order**, one at a time, until back on schedule:

| # | Dropped item | Where it lives | Note |
|---|---|---|---|
| 1 | Rule 5c test: approve vs cancel race | step 18 | Rules 5a and 5b stay; they are rule 5. |
| 2 | Admin allocation edit vs approval race test | step 19 | The single-statement upsert and the `ALLOCATION_BELOW_USED` test stay. |
| 3 | America/New_York per-company timezone test | step 9 | Probably already written by hour 30; if so, keep it. The IST-midnight test is not cuttable. |
| 4 | Admin all-requests **page** | step 25 | `GET /admin/requests` **stays**. The brief says the admin sees every request; the screen list only requires the balances table. |
| 5 | GitHub Actions | step 27 | Tests still run locally against Docker Postgres. |

- **Everything else is must-have.** That includes every test for rules 3, 4, 5a, 5b, 6 and 7, the login enumeration tests, the seed invariants, and the live deploy.
- Anything dropped goes into README "Known limitations" and "What I'd do next", with one line on why.
- Cut items are marked ✂️ in the steps below.

---

## Phase 0 — Pre-flight

### 1. Repo init with spec first
- Commit `SPEC.md`, `PLAN.md`, `AGENT_LOG.md` (holding the planning conversation so far), a stub `README.md`, `.gitignore` and `.editorconfig`.
- **Done when:** the first commit contains no code.
- **Commit:** `docs: spec and build plan`

### 2. Neon btree_gist pre-flight
- In the Neon SQL editor, run `CREATE EXTENSION IF NOT EXISTS btree_gist;`.
- Create a throwaway table with an `EXCLUDE USING gist (k WITH =, daterange(a,b,'[]') WITH &&)` constraint. Insert two overlapping rows and confirm the second fails with `23P01`. Drop the table.
- Record the result in SPEC §12.
- **Done when:** the result is recorded. If it fails, stop and revisit D4.
- **Commit:** `docs: confirm btree_gist exclusion constraints work on Neon`

---

## Phase 1 — Walking skeleton, deployed

### 3. Server scaffold, config and route registry with health check
- `server/`:
  - TypeScript strict, ESLint, Vitest (sequential files)
  - npm scripts: `dev`, `build`, `typecheck`, `test`
- `docker-compose.yml`:
  - `db` on 5432
  - `db-test` on 5433 with tmpfs storage
- `config.ts`: typed env. Exits at startup without `JWT_SECRET` (≥32 chars).
- `createApp({ clock })` with the minimal `RouteDef` registry from SPEC §6.0. Its only route for now is `GET /api/health`, returning `{status:'ok'}` without a DB check yet.
- CORS allowlist from `CORS_ORIGIN`.
- `.env.example` with every env var from SPEC §11.3.
- Tests: startup fails without the secret; health returns 200; a disallowed origin gets no CORS headers.
- **Commit:** `chore: scaffold server, config, registry and health route`

### 4. Prisma schema and hand-edited migration
- Schema per SPEC §2:
  - enums and all four tables
  - uniques, indexes, `@@map`
  - composite relations where Prisma accepts them
- `migrate dev --create-only`, then append raw SQL:
  - `btree_gist`
  - `leave_request_no_overlap`
  - all CHECKs
  - any composite FKs Prisma refused
- Prisma's direct-URL setting for migrations.
- Health now runs `SELECT 1` and returns 503 when the DB is down.
- **Done when:** `\d+ leave_request` shows the exclusion constraint and every CHECK from SPEC §2.5, and health reports `db: ok` locally.
- **Commit:** `feat(db): schema, exclusion constraint and check constraints`

### 5. Bare web scaffold
- `web/` with Vite, React and TypeScript.
- A single page that calls `${VITE_API_URL}/api/health` and shows the result.
- Vite dev proxy for `/api`.
- `vercel.json` with the SPA rewrite.
- **Done when:** the local page shows `api ok / db ok`.
- **Commit:** `feat(web): bare scaffold with health check page`

### 6. Deploy the skeleton
- **Render:**
  - Web service from `server/`.
  - Build: install → `prisma generate` → `prisma migrate deploy` → `tsc`.
  - Env: `DATABASE_URL` (Neon pooled), `DIRECT_URL` (Neon direct), `JWT_SECRET`, `CORS_ORIGIN`, `BCRYPT_COST`.
- **Vercel:** project from `web/` with `VITE_API_URL` set to the Render URL.
- Turn on auto-deploy from `main` for both.
- Verify:
  - The Neon tables exist with the exclusion constraint (`\d+` against Neon).
  - The Vercel page shows `api ok / db ok`.
  - A request from another origin is refused by CORS.
- Record any production surprises in `AGENT_LOG.md`.
- **Done when:** the live Vercel URL shows `db ok` and the skeleton redeploys after a push to `main`.
- **Commit:** `chore: deploy skeleton to render, vercel and neon`

---

## Phase 2 — DB correctness

### 7. Test harness, constraint tests and `checkInvariants`
- `assertSafeToReset(url)`: allows only `localhost`, `127.0.0.1`, `::1`, `db` and `db-test`.
- `resetDb()` calls `assertSafeToReset` first.
- `migrate deploy` to `db-test` once per run.
- A minimal two-company fixture builder.
- `src/db/invariants.ts` with `checkInvariants()` per SPEC §9.
- Raw-SQL tests proving the DB refuses bad data:
  - an overlap → `23P01`
  - REJECTED and CANCELLED rows don't block a new request
  - different leave types still conflict
  - `used > allocated` is rejected
  - `decided_by = employee` is rejected
  - a cross-company `employee_id` is rejected
  - an admin with a manager is rejected
  - a cross-year request is rejected
  - `working_days` of 0 or 31 is rejected
- Tests for the guard and the invariants:
  - `assertSafeToReset` rejects a Neon-style URL.
  - `checkInvariants` catches a hand-planted APPROVED row with `used_days` left at 0.
- **Commit:** `test(db): constraint tests, reset guard, invariant checker`

### 8. Spike: 23P01 / 23514 shapes and `pgErrors.ts`
- Throwaway script:
  1. Insert an overlapping request via Prisma `create()`, then via `$queryRaw`.
  2. Log the class, `code`, `meta`, `message` and `cause` of each error.
  3. Repeat for the balance CHECK violation.
- Record the findings and the chosen insert path in SPEC §12. Delete the script.
- Implement `isOverlapViolation` and `isBalanceCheckViolation`, with the true/false tests from SPEC §6.4.
- **Commit:** `feat(db): pg error detection with isOverlapViolation, from spike findings`

### 9. Date helpers
- `parseDate`, `formatDate` (UTC getters only), `workingDays`, `todayIn(tz, clock)`, `sameYear`.
- Unit tests:
  - Fri–Mon = 2
  - weekend-only = 0
  - single day = 1
  - 30 vs 31 working days
  - `2026-02-30` is rejected
  - at `19:00Z` the date in Asia/Kolkata is the next day
  - America/New_York differs from Asia/Kolkata at the same instant ✂️ cut #3
- This step comes before the seed because the seed uses `workingDays` and `todayIn`.
- **Commit:** `feat: date and working-day helpers`

### 10. Seed (insert-only, ledger-consistent)
- Implements SPEC §11.1 in full:
  - one transaction
  - refuses a non-empty DB
  - APPROVED samples increment `used_days` in the same transaction
  - `decided_by` is the direct manager
  - no overlapping samples
  - dates stay inside the current year, with the backdated-SICK fallback
  - `checkInvariants()` at the end, rolling back on failure
  - bcrypt with `BCRYPT_COST`
- Add the `db:reset` script (guarded, local only) and `check:invariants` (read-only).
- Tests on `db-test`:
  - the seed passes the invariant check
  - every user's balance cards would show `used` equal to the sum of their approved samples
  - a second run exits non-zero and changes nothing
  - `db:reset` refuses a Neon-style URL
- **Commit:** `feat(db): insert-only seed that keeps the leave ledger consistent`

---

## Phase 3 — API

Each step auto-deploys. Steps marked **live** add a smoke check against Render.

### 11. Error handling and Zod validation
- `AppError` plus the error-code catalogue from SPEC §6.3.
- Error middleware using `pgErrors`. No stack or SQL in responses.
- `validate()` middleware: `.strict()` on params, query and body; 400 shape with `details`.
- 404 fallthrough.
- Tests: an unknown-route 404 shape; a validation error shape; an unknown key → `UNRECOGNIZED_KEYS`.
- **Commit:** `feat(api): error handling and zod validation`

### 12. Auth, constant-work login and role gate — **live**
- `POST /auth/login` per SPEC §4:
  - `DUMMY_HASH` computed at startup with `BCRYPT_COST`
  - exactly one `bcrypt.compare` per login
  - byte-identical 401s
- JWT: HS256 pinned, 8h, `sub` only.
- `authenticate` reloads the user from the DB on every request.
- `requireRole` reads `roles` from the registry.
- `GET /me`.
- Tests:
  - login success and failure
  - expired token, wrong secret, `alg: none`, token for a deleted user
  - **enumeration:** the compare spy is called once for an unknown email and once for a wrong password, and the bodies are identical
  - **enumeration timing:** the median-ratio check from SPEC §9
  - **role-gate loop** over the registry
- **Live:**
  1. Seed Neon once from your machine (`npm run seed` with `DIRECT_URL`), then run `npm run check:invariants` against Neon.
  2. `curl` a login on Render for one user from each company.
- **Commit:** `feat(api): constant-work login, jwt auth, registry-driven role gate`

### 13. Tenant-scoped repositories and visibility
- Every repo function takes `ctx`. Every `where` has `companyId`. Lookups by ID use `findFirst` with `companyId` in the same query, with no load-then-compare (SPEC §5).
- `canSee(ctx, request)`. Handlers never import Prisma.
- Visibility tests covering owner, direct manager, other manager, admin and other company.
- **Done when:** a review of every repo query is logged in `AGENT_LOG.md`.
- **Commit:** `feat(api): tenant-scoped repositories and visibility rules`

### 14. Employee read endpoints
- `GET /me/balances` (with pending and available), `GET /me/requests`, `GET /requests/:id`.
- Tests: the balance arithmetic with pending; seeded approved samples show non-zero `used`; another manager's request → 404.
- **Commit:** `feat(api): balances and request read endpoints`

### 15. Apply for leave (rule 3 and submission checks)
- `POST /me/requests` per SPEC §6.2. Inserts use the path chosen in step 8.
- Tests:
  - **Rule 3:**
    - overlap with PENDING or APPROVED → 409
    - overlap with REJECTED or CANCELLED → 201
    - a different leave type still conflicts
    - a same-day touch → 409; adjacent ranges → 201
    - **two identical requests fired concurrently → one 201, one 409, one row (10 rounds)**
  - Backdating, including the IST-midnight case with a fixed clock.
  - All Zod limits.
  - `NO_BALANCE_FOR_YEAR`; `INSUFFICIENT_BALANCE` including pending days.
- **Commit:** `feat(api): leave submission with overlap and balance checks`

### 16. Cancel
- `POST /requests/:id/cancel`, as a guarded `UPDATE`.
- Tests:
  - the owner cancels → 200, balance untouched
  - the manager tries to cancel → 403 `NOT_REQUEST_OWNER`
  - cancelling a non-pending request → 409
- **Commit:** `feat(api): cancel pending request`

### 17. Approvals list and reject
- `GET /approvals` lists direct reports only.
- `POST /requests/:id/reject` with a required comment.
- **Rule 6 tests (reject):**
  - own request → 403 `SELF_APPROVAL_FORBIDDEN`
  - the other manager's report → 404
  - admin on an employee's request → 403 `NOT_DIRECT_MANAGER`
  - admin on a manager's request → 200
  - an employee calling reject → 403
- **Commit:** `feat(api): approvals list and reject`

### 18. Approve (rules 4, 5 and 6) — **live**
- The transaction from SPEC §7.1.
- Tests, each followed by `checkInvariants()`:
  - **Rule 4:** submit, lower the allocation (direct DB write until step 19), approve → 409. The request stays PENDING and `used` is unchanged.
  - **Rule 5a:** the same request approved twice concurrently → exactly one 200, `used` moves once (10 rounds).
  - **Rule 5b:** two requests whose combined days exceed the balance, approved concurrently → one 200, one 409, and `used ≤ allocated` (10 rounds).
  - **Rule 5c:** approve vs cancel concurrently → exactly one wins. ✂️ cut #1
  - **Rule 6:** the same matrix as step 17, for approve.
- **Live:** approve one seeded pending request through `curl` on Render, then run `check:invariants` against Neon.
- **Done when:** the concurrency tests pass 3 full runs in a row.
- **Commit:** `feat(api): atomic approval with guarded balance deduction`

### 19. Admin endpoints
- `GET /admin/requests`, `GET /admin/balances`, `PUT /admin/balances/...` (single-statement upsert).
- Switch the step 18 rule-4 test to use this endpoint.
- Tests:
  - `ALLOCATION_BELOW_USED`
  - `USER_HAS_NO_BALANCES`
  - an upsert creates a missing year
  - an allocation edit racing an approval passes the invariants ✂️ cut #2
- **Commit:** `feat(api): admin requests list and balance management`

### 20. Rule 7 suite (registry loop) — **live**
- A probe map keyed by `` `${method} ${path}` ``. A route with a `:param` and no probe fails the test.
- For each probe:
  - A company-A actor sends a company-B ID → 404.
  - The response is byte-identical to the response for a random UUID.
  - For mutating routes, the B row is unchanged.
- Every list endpoint returns zero rows from B. `companyId` in a body or query → 400.
- Temporarily add a dummy `/:id` route, confirm the suite fails, revert it, and log the check.
- **Live:** `curl` with a company-A token on a company-B request ID → 404 on Render.
- **Commit:** `test(api): cross-tenant isolation over the route registry`

### 21. API review pass
- Re-read every repo function and raw SQL statement against SPEC §5 and §7.
- Search for any write of `status = 'APPROVED'` outside the approve transaction and the seed.
- Run the full suite 3 times. Log each fix with its reason.
- **Commit:** `chore(api): review fixes`

---

## Phase 4 — UI

Each step is live on Vercel as soon as it's pushed.

### 22. API client, user switcher and navigation
- A typed client for the SPEC §6.1 DTOs.
- Token storage.
- A switcher listing the static seed users from both companies.
- Role-based navigation.
- An error-code → message map.
- **Commit:** `feat(web): api client, user switcher, role nav`

### 23. My leave page
- Balance cards, the apply form with a working-days hint, and the request list with Cancel on pending requests.
- **Commit:** `feat(web): my leave page`

### 24. Approvals page
- Pending list, Approve (optional comment) and Reject (required comment), with 409s shown inline.
- **Commit:** `feat(web): approvals page`

### 25. Admin pages
- Balances table with a year selector and inline edit.
- All-requests list with a status filter. ✂️ cut #4 (page only; the endpoint stays)
- **Commit:** `feat(web): admin balances and all-requests pages`

### 26. Walkthrough on the live site
- Every role in both companies, on the live Vercel URL.
- Two tabs approving the same request.
- `check:invariants` against Neon afterwards.
- **Commit:** `fix(web): walkthrough fixes`

---

## Phase 5 — Final checks and docs

### 27. Production hardening (optional)
- GitHub Actions running the test suite against a Postgres service container. ✂️ cut #5
- Confirm the Render and Vercel env vars match `.env.example`.
- If the live data got messy during testing: reset the Neon branch deliberately in the console, re-seed, then run `check:invariants`.
- **Commit:** `chore: ci and production checks`

### 28. README and log
- **Setup:** Docker, env, migrate, seed, test, dev.
- **Design decisions:** a summary that links to SPEC.
- **Known limitations:**
  - the admin can't apply for leave
  - the submission balance check is soft
  - no RLS
  - free-tier cold starts on Render and Neon
  - the other out-of-scope items in SPEC §10
- **Next steps:** RLS; cancelling approved leave with a refund; holidays and half-days; an audit log; pagination.
- Live links. Finalize `AGENT_LOG.md`.
- **Commit:** `docs: readme, live links and agent log`
