# Developer Growth Roadmap

## Tier 0 — SaaS/production architecture (added 2026-08-05, from full codebase audit)

Found via actually reading `c:\pms\backend`, not guessing. Ordered by real risk in this codebase.

| # | Gap found | Where | Resource |
|---|-----------|-------|----------|
| 0a | Tenant isolation not structurally enforced (relies on every query remembering `where: tenantId`) | `common/middleware/tenant.middleware.ts` | AWS SaaS Factory — tenant isolation models (silo/pool/bridge) |
| 0b | Only 1 migration file for 455 source files — schema likely not tracked incrementally | `database/migrations/` | TypeORM migrations docs; treat `synchronize: true` as dev-only from now on |
| 0c | No rate limiting on any endpoint (incl. login) | — | `@nestjs/throttler` docs |
| 0d | No health-check endpoint for LB/uptime monitoring | — | `@nestjs/terminus` docs |
| 0e | CORS reflects any origin with credentials (`origin: true`) | `main.ts` | OWASP CORS misconfiguration entry |
| 0f | No env validation at boot | — | `@nestjs/config` + Joi schema docs |
| 0g | Not containerized despite a past session about it — deploy is raw scp+PM2 | `.github/workflows/deploy-prod.yml` | Docker docs; "Twelve-Factor App" (12factor.net) — factor VI/X |
| 0h | No error tracking / metrics beyond log files | — | Sentry free tier docs |
| 0i | Frontend has ~3 test files total | `NAVISHR-Dashboard/src` | (use same testing habit as backend, just ported to frontend) |

Foundational reading for this tier: **The Twelve-Factor App** (12factor.net, short, free) —
covers config validation, containerization, logs-as-streams; most gaps above are literally
named factors in it.

## Placement module tenant-isolation pass — 2026-08-06

Full audit of the placement module, controller-by-controller, service-by-service.
Pattern found repeatedly: a `tenantId` or caller-supplied ID would arrive at a service
method and simply not get used in the query. Fixed by (a) a batch-ownership guard
(`assertBatchOwnership`, mirrored in exam-bookings) for any endpoint taking a `batchId`
from the URL, and (b) a shared `CandidateAccessService.assertOwnership(tenantId,
candidateId)` — since `CandidateProfile` has no `tenantId` column, ownership is proven by
joining through `candidate.batch.tenantId`.

**Fixed and verified (type-check + baseline-compared test run, no regressions):**
tasks.service.ts (original bug), batches.service.ts (5 sub-resource endpoints), all 5
candidate-tab services, vendors/accommodations/exam-bookings/reconciliations/travel in
concierge. budget.service.ts and disbursements.service.ts were already correct.

**Found, NOT fixed — real prerequisites, not urgent today:**
1. **`CandidateProfile` has no tenant identity until batch enrollment.** Traced all the
   way to the public `exam-player` app, which sends no tenant context at registration.
   `assessment/students` module (`findAllCandidates`, `findPool` in onboarding) lists
   candidates with zero tenant filter. Confirmed with the user: only NAVISHR currently
   uses these modules, so this is not live cross-tenant exposure today — but it is a
   **hard blocker before any second tenant gets the placement/assessment modules
   enabled.** Needs: tenant context captured at registration (or a deliberate decision
   that the pool stays shared and pre-enrollment access is gated some other way),
   `tenantId` added to `candidate_profiles`, backfilled, enforced.
2. **`PgAccommodation` has no `tenantId` column** — unlike its sibling concierge
   entities (Vendor, VendorPayment), this looks like a plain oversight, not a
   deliberate design. Lower risk than #1 (no public app involved), but `findAll`/
   `create` in accommodations.service.ts are still unscoped until this is added.
   Service code is written to accept `tenantId` already, so wiring it in once the
   column exists is a small change.

Both are noted here rather than fixed silently — they're schema changes on a live
table, and #1 in particular reaches into a module and a public app outside what was
asked. Revisit before onboarding a second tenant onto placement/assessment.

## Update 2026-08-06 — candidate tenant identity, backend half built

Decided: per-company subdomains for exam-player (`client-a.exams.navishr.com`),
hosted on Amplify (wildcard subdomain support, one deployment serves all companies —
no redeploy per client). Reuses `TenantPublicController.resolve` and
`TenantService.findBySubdomain`, which already existed for the main dashboard's login
flow — no new lookup endpoint needed.

**Built and verified (migration tested up AND down against local DB):**
- `candidate_profiles.tenant_id` — nullable uuid column + index, migration
  `1785992700000-AddTenantIdToCandidateProfile.ts`.
- `CreateCandidateProfileDto.tenantSubdomain` (optional) — what exam-player will send;
  NOT a raw tenantId, by design (public endpoint, client input isn't trusted directly).
- `StudentsService.createCandidateProfile` resolves the subdomain server-side via
  `TenantService.findBySubdomain` and stamps `tenantId` on creation. Rejects with a
  clear error if the subdomain doesn't resolve to an active tenant (bad/stale link);
  still allows no subdomain at all during rollout (existing behavior, unchanged).

**Still open:**
1. **Amplify wildcard domain setup** (`exams.navishr.com`, `*` subdomain rule) — done
   in the AWS console, not something I can do from here.
2. **`exam-player` frontend** — needs to read `window.location.hostname`, extract the
   subdomain, call `GET /tenants/public/resolve?subdomain=...` for branding, and send
   the subdomain (not tenantId) at registration. Not started.
3. **Backfill existing rows** — all current candidates belong to NAVISHR; needs a
   manually-reviewed one-off `UPDATE candidate_profiles SET tenant_id = '<navishr-id>'
   WHERE tenant_id IS NULL` run against staging/prod once, NOT baked into the migration
   (tenant id differs per environment).
4. **Enforce `NOT NULL`** — separate follow-up migration, only after #3 has actually
   run in that environment.
5. **`findPool`/`findAllCandidates`/etc. still don't filter by tenantId** — the column
   existing doesn't retroactively fix the read-side leak from before; those queries
   still need the `WHERE tenantId = :tenantId` added once backfill (#3) makes it safe
   to enforce without breaking access to legacy null-tenant rows.

## Update 2026-08-06 (cont.) — exam-player frontend done

Decided: no wildcard subdomain — each client company's subdomain gets added manually
in Amplify's Domain Management as they onboard. Reject anything unrecognized instead
of defaulting it, EXCEPT relaxed in dev/preview builds so local testing isn't blocked.

Built in `exam-player` (type-checked + full production build verified clean):
- `src/lib/tenant.ts` — `resolveTenantSubdomain()`, three-way classification:
  `client-a.exams.navishr.com` → `"client-a"`; bare `exams.navishr.com` → `"navishr"`
  (matches backend's `DEFAULT_TENANT_SUBDOMAIN`, must stay in sync); anything else →
  `"unrecognized"` in production, falls back to `"navishr"` in dev
  (`import.meta.env.DEV`) so localhost/Amplify-preview testing still works.
- `src/components/TenantGate.tsx` — wraps the whole app; renders a "Link Not
  Recognized" page instead of the exam/registration flow when unrecognized, in prod.
- `RegisterCandidateDto.tenantSubdomain` + `StudentExamPlayerPage` now sends the
  resolved subdomain at registration (never a raw tenantId — backend re-resolves it
  itself, per the trust-boundary reasoning worked through earlier).

**Still open, unchanged from before:** Amplify custom-domain setup (console, not
code), backfill existing candidates to NAVISHR's real id, enforce `NOT NULL`, and the
read-side queries (`findPool` etc.) still need `WHERE tenantId` added once backfilled.

Also noted in passing: `exam-player`'s `package.json` had two dependencies
(`@tensorflow-models/blazeface`, `react-icons`) that were declared but never
installed in this dev environment — fixed via `npm install`, unrelated to anything
built today, but worth knowing `node_modules` here had silently drifted from
`package.json` before this.



Personal reference list — not a to-do-all-at-once list. Pace: one item at a time,
one chapter/resource in one sitting, one ADR after each. See "How to use this"
at the bottom.

## Tier 1 — Do these first (directly fixes your biggest recurring bugs)

| # | What | Resource | Why it's Tier 1 |
|---|------|----------|------------------|
| 1 | Systems thinking foundations | **"Designing Data-Intensive Applications" (Kleppmann)** — Part 1, ch. 1–3 | Explains reliability/scalability/maintainability tradeoffs, data models, and storage engines. Directly explains the Postgres/TypeORM decisions you live with daily. |
| 2 | Writing ADRs | No book — just the habit (format below) | Converts "I feel like I know nothing" into an actual personal tradeoff library, built from decisions you already made. |
| 3 | Permission/RBAC design | NestJS docs → "Authorization"; Auth0 blog "RBAC vs ABAC" | Your #1 recurring bug source (18 sessions across 2.5 months). Fixes the "patch one endpoint at a time" pattern in `permissions.guard.ts`. |
| 4 | Testing | NestJS docs → "Testing" (Jest) | Zero tests across 183 conversations. Would have caught the repeat bugs (department label, permission bypass, forgot-password rework). |
| 5 | TypeScript depth | TS Handbook — "Everyday Types" + "Narrowing" chapters; "Type Challenges" repo (GitHub) | Your single largest failure category (62/541 failed commands, ~11.5%). |

## Tier 2 — Next, once Tier 1 feels boring/solid

| # | What | Resource | Why |
|---|------|----------|-----|
| 6 | API design | Google "API design guide" (cloud.google.com/apis/design); read Stripe's API docs as a reference | You've made REST decisions (PATCH vs PUT) once well — generalize it. |
| 7 | Concurrency & consistency | DDIA Part 2; "Distributed Systems for Fun and Profit" (free online book, short) | Invisible until it bites — double-bookings, race conditions in concierge/disbursement flows. |
| 8 | Security fundamentals | OWASP Top 10 (owasp.org) | Permanent checklist, read once, carry forever. |
| 9 | Architecture/code judgment | "A Philosophy of Software Design" (Ousterhout) | Coupling/cohesion, when to abstract vs not — guards against over-engineering once patterns start clicking. |
| 10 | General engineering judgment | "The Pragmatic Programmer" (Hunt & Thomas) | Short, not code-heavy, about judgment calls broadly. |

## Tier 3 — Background / map, not urgent reading

| # | What | Resource | Why |
|---|------|----------|-----|
| 11 | Full concept map of backend engineering | roadmap.sh/backend | Use as a checklist against your own project — which boxes has NAVISHR touched, which haven't you needed yet. |
| 12 | Your actual framework, read as a book | NestJS official docs, start to finish, once | You use it daily but only ever search it piecemeal. Retroactively explains a lot of "why is this structured this way." |
| 13 | Git branching discipline | "A successful Git branching model" (nvie.com) | You've already improved at git mechanics (June onward) — this fixes the remaining branch-sprawl habit. |

## Habits to run in parallel (not "study," just do these going forward)

- **One ADR per week** on a decision you already made without deliberating (auth strategy, DB choice, module boundaries). Format:
  - *Context:* what problem were we solving
  - *Decision:* what we did
  - *Alternatives considered:* what else could've worked
  - *Tradeoff:* what we gave up
- **Design doc before code** for any new module — 10 lines: entities, endpoints, request/response shapes, edge cases. (You did this once, for the Placement API — Jul 28 session — make it the default, not the exception.)
- **Refactor pass ~1 week after a feature ships** — 30–60 min revisit while it's cheap, not after it's broken again.
- **Consolidate workspace** — stop keeping `-backup` / `restored_src` copies of projects; use git tags/branches instead.

## How to use this list

Don't parallelize. Go top to bottom, one row at a time. For a book, that
means one chapter in one sitting — not spread thin across a week. After
each item, write the ADR it maps to before moving to the next row. If an
item feels solid and boring by the time you get through it, that's the
signal to move to the next one — not before.

Started: 2026-08-05
