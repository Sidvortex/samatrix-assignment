# Agent log

How this project was built with an AI agent, and where the agent's output was reviewed and corrected.

- **Agent:** Claude (claude.ai chat, Claude Opus 5.5)
- **Format:** my prompts are copied verbatim. The agent's responses are summarized, because the full outputs live in the files they produced (`SPEC.md`, `PLAN.md`). Times are IST.
- Each entry ends with **Review**: what I accepted, what I changed and why.

---

## Session 1 — Spec and plan, before any code (2026-10-29)

### 1.1 · 07:50 — Ask for a spec, with questions first

> I’m doing a 48-hour take-home build task: [Build task: Leave requests for Comsky HRMS
> Time limit: 48 hours from when you start
> Stack: your choice. The JD favours TypeScript, Node (Express or Fastify), PostgreSQL with Prisma, and React.
> The feature
> Several companies (tenants) use the same HRMS. Employees request leave, managers approve or reject it, and balances update correctly.
> Users and roles
>
> * Employee: applies for leave, sees their own requests and balance, and can cancel a request that’s still pending.
> * Manager: sees requests from their direct reports and approves or rejects them with a comment.
> * Admin: sets each employee’s yearly balance per leave type and sees every request in their own company.
>
> Seed data: 2 companies, each with 1 admin, 2 managers and 4 employees. No signup flow is needed. A simple login or a user switcher is enough, as long as the API still enforces roles.
> Rules
>
> 1. Leave types are Casual, Sick and Earned, each with its own balance.
> 2. A request has a start date, an end date, a type and a reason. Weekends don’t count toward days used.
> 3. Overlapping requests from the same employee are rejected, unless the existing request was rejected or cancelled.
> 4. A request that would push the balance below zero is rejected at approval time, not just at submission.
> 5. Approval and the balance deduction must happen atomically. Two managers approving at the same moment must not double-deduct.
> 6. A manager can’t approve their own leave, and can’t act on requests from employees who aren’t their direct reports.
> 7. No user can ever read or change data from another company, even by guessing IDs.
>
> Screens
>
> * Employee: a “My leave” page with balances, an apply form and a request list with statuses.
> * Manager: an “Approvals” page with pending requests from their team and approve/reject buttons.
> * Admin: a balances table for the whole company.
>
> Submit
>
> * A repo link with a README covering setup, design decisions, known limitations and what you’d do next.
> * Your prompts or agent log.
> * Tests that cover at least rules 3, 4, 5, 6 and 7.
> * A live link, if you can deploy it.
>
> How it’s judged
>
> * Correct data and authorisation over UI polish.
> * Whether your spec came before the code.
> * Whether your log shows you reviewing and correcting the agent, not just accepting its output.]. Stack: TypeScript end to end, Node + Express, PostgreSQL + Prisma, React + Vite. Don’t write any code yet. First help me write SPEC.md covering the data model (tables, keys, indexes), the API contract (endpoints, request/response shapes, error codes), how tenant isolation is enforced, how approval stays atomic under concurrent requests, edge cases, and what’s out of scope. Ask me about anything ambiguous before assuming.

**Agent:** wrote no spec yet. It raised 6 ambiguities, each with a recommendation:
1. who approves a manager's leave
2. requests that cross a year boundary
3. whether pending days count against the balance at submission
4. how to enforce rule 3 (overlaps) under concurrency
5. how deep tenant isolation should go
6. whether backdated requests are allowed

It also listed the defaults it would take otherwise: JWT auth, UUIDs, `DATE` columns, statuses, 404-vs-403 semantics, a guarded-`UPDATE` design for atomic approval, and the out-of-scope list.

**Review:** the questions were a good start. My answers are in 1.2.

### 1.2 · 07:54 — Answers, one override, and a gap the agent missed

> 1 - a. admin approves the managers. admin can’t apply for leave, just put that in the readme limitations
> 2 - a, reject it
> 3 - a, approval re-checks anyway so fine
> 4 - a, but map the postgres error code 23P01 to 409 OVERLAPPING_REQUEST explicitly. and add a test where two identical requests get submitted at the same time, only one should go through
> 5 - a. put RLS in the readme as a next step. for tests though I want one that loops over every endpoint that takes an id, passes an id from the other company and checks it’s a 404. one example test isn’t enough
> 6 - changing this one. only sick leave can be backdated. casual and earned have to start today or later, otherwise BACKDATE_NOT_ALLOWED
> also you missed something. what’s “today”? dates are plain DATE but the server’s on UTC, so for someone in india between midnight and 5:30am “today” is actually yesterday. add a timezone column on company, default Asia/Kolkata, and work out today using that
> two more things:
>
> * tests should run on a real postgres in docker, not sqlite or mocks. the exclusion constraint and the concurrency tests don’t prove anything otherwise
> * zod validation on every endpoint. reason required, max 500 chars, a single request can’t go over 30 working days, reject unknown fields
>
> rest of the defaults are fine. jwt secret from env, don’t hardcode it. go ahead and write SPEC.md

**Agent:** wrote `SPEC.md` v1.

**Review:**
- **Overrode** the backdating recommendation. Allowing every type to be backdated is wrong; only sick leave is.
- **Caught a gap the agent missed.** It never defined "today". With a UTC server, users in India get the wrong date between 00:00 and 05:30 IST. Fixed with a per-company timezone and an injectable clock.
- **Tightened the tests.**
  - Added an explicit `23P01` → 409 mapping and a concurrent identical-submission test.
  - Replaced a single example isolation test with a loop over every ID endpoint.
  - Required real Postgres for tests, because SQLite or mocks wouldn't exercise the exclusion constraint or the row locks.
- Added the Zod limits and the env-only JWT secret.

### 1.3 · 08:03 — Push back on four of the agent's calls

> mostly good, few things
> the rule 5 setup works for the insufficient balance case, keep it. but also keep the plain double-approval test, same request approved twice at the same time (two tabs, double click), exactly one goes through and used only moves once. that’s what rule 5 actually says
> for 23P01 don’t leave it as “either way” in the spec. before writing the service, do a quick check: create an overlapping row through prisma and see what error actually comes back. then pick one approach, and put the detection in one function like isOverlapViolation(err) with its own test
> the router introspection idea is good but reading express internals is fragile, it changed in express 5. instead keep one exported route list (method, path, roles), use it to register the routes, and have the test loop over the same list. then a new route can’t skip the test
> hosting: neon for postgres since the free tier doesn’t expire, render for the api, vercel for the frontend. first thing, run CREATE EXTENSION btree_gist on neon to confirm it works before we depend on it
> update the spec with these and then give me a build plan in small steps, each one ending in a commit. db and constraints first, then api, then ui, deploy last. no code yet

**Agent:** `SPEC.md` v1.1 and `PLAN.md` v1. It pointed out the double-approval test already existed as rule-5 test (a), and made it explicit as its own test.

**Review:**
- **Rejected "either way" on 23P01.** A spec that leaves the error-detection path open isn't a spec. A spike now decides one path, and detection is isolated in `isOverlapViolation` with its own tests.
- **Rejected router introspection.** Express 5 changed its internals. A shared route registry now drives route registration, the role-gate test and the tenant-isolation test.
- **Decided hosting**, with a btree_gist pre-flight before anything depends on it.

### 1.4 · 12:12 — Review of the plan: four bugs

> The seed will create broken data. It inserts sample APPROVED requests directly, but the plan never says to increase used_days for them. The balance cards would show 0 used next to an approved leave. Also, the seed truncates every table, so accidentally running it twice against Neon would wipe the live demo. also, Login leaks which emails exist. For an unknown email it skips bcrypt and returns instantly, while a wrong password takes 100ms. Same 401, different timing that’s exactly the kind of bug in the JD is about. Deploying last is the riskiest part of the plan. Neon pooling, Render builds, CORS and Prisma on Render all tend to break in production If that happens at hour 44 you have nothing live. Deploy a bare skeleton early instead.

**Agent:** `SPEC.md` v1.2 and `PLAN.md` v2.
- **Ledger invariant:** `used_days` must equal the sum of approved working days. A read-only `checkInvariants()` checks it, and the seed is written to satisfy it.
- **Insert-only seed** that refuses a non-empty database. A local-only `db:reset`, with the same guard extended to the test harness's truncate.
- **Constant-work login:** one bcrypt compare against a dummy hash of the same cost, plus a spy test and a coarse timing test.
- **Same-code-path 404s** for cross-tenant IDs.
- **Walking skeleton** deployed at step 6, with migrations in the Render build and live smoke checks after that.

**Review:** all four were real bugs in the agent's plan.
- **Seed:** wrote APPROVED rows without updating `used_days`, which breaks the data model on first load.
- **Truncate:** could destroy the live demo.
- **Login timing:** leaked which emails exist despite the identical 401.
- **Deploy last:** pushed all production risk to the final hours.

The agent extended two of the fixes past what I asked for. I checked both and kept them:
- The test truncate had the same Neon-wipe risk as the seed.
- Cross-tenant 404s had the same timing-branch risk as login.

### 1.5 · 12:18 — Cut line, then start

> also add a cut line to PLAN.md. if I’m behind at hour 30, these get dropped in this order: rule 5c approve vs cancel test, the admin edit vs approval race test, the america/new_york timezone test, admin all-requests page, github actions. everything else is must-have
> update SPEC changelog + PLAN, then commit step 1 and we start step 2

**Agent:**
- Added the cut line to `PLAN.md` at the hour-30 checkpoint, which means step 21 not done.
- Marked each cuttable item ✂️ in its step. Noted that cutting the admin all-requests page keeps `GET /admin/requests`, since the brief requires the admin to see every request.
- Added `SPEC.md` v1.3.
- While preparing this commit, found and fixed two stale lines in its own earlier output: §6.4 still pointed the spike at the old step 6 instead of step 8, and §12 had a duplicated "pending" row.

**Review:** _(to fill in)_

---

## Session 2 — Build

_(Entries start at PLAN step 2.)_
