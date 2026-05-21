# Dev-Ready Task — P1.9 · Demo seed harness

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.9**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md#pre-flight-25`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready (3 architectural calls to confirm before code lands — see "Decisions to confirm" below)

## Goal

Turn the deployed `caseflow.musto.io` from "registration + an empty dashboard" into a **drivable demo** with realistic data — three tenants, ~10 users per tenant across all roles, ~35 cases per tenant in varied statuses. The seed runs locally via `yarn db:seed:demo` and against the deployed RDS via `scripts/dev.sh demo:seed --target=rds`. Both entry points are idempotent — re-running wipes the seed-managed rows and recreates them.

After P1.9 ships, Phase 1 closes at 9/9 and the project becomes credibly demo-ready for the LinkedIn audience that's now arriving at the deployed URL.

## Why this architecture (the 90-second mental model)

Three properties make the seed worth its weight rather than just a one-off script:

**Determinism.** Same `SEED_VERSION` → byte-identical output every run. Case A's UUID is the same UUID every run, every environment. No `Math.random()`. UUIDs are derived from `hash(tenant_slug + entity_kind + entity_number + SEED_VERSION)`. This means screenshots in the content series can reference specific cases by ID and stay accurate when readers spin up their own copy.

**Idempotency.** Running the seed twice is safe. Running it ten times is safe. The reset path (`demo:reset`) is the same as `demo:seed` because deterministic UUIDs let the seeder identify which rows it owns and which it should leave alone — the seeder deletes rows by exact ID match before recreating.

**Multi-tenancy proof.** Three tenants in the dataset (not one) so the multi-tenant isolation invariant is visible at first click. A demo viewer logging in as a user from `acme-services` literally cannot see `globex-support` data — that's the receipt.

## Affected areas

### Backend — new files

- `backend/src/seed/seed.module.ts` — NestJS module wiring (TypeOrmModule.forFeature on every seedable entity).
- `backend/src/seed/seed.service.ts` — orchestrator. Public method `seedDemo()`. Internally calls per-entity factories in order: tenants → users → cases.
- `backend/src/seed/factories/tenants.factory.ts` — produces the 3 tenant rows. Slugs: `abc-company` (existing default), `acme-services`, `globex-support`.
- `backend/src/seed/factories/users.factory.ts` — produces ~10 users per tenant. Roles: 1 ADMIN, 3 ENGINEER, 2 AGENT, 4 CUSTOMER. One PENDING_APPROVAL per tenant (an extra user; the others are ACTIVE). Passwords are bcrypt-hashed from a known plaintext (`demo123!`) — same plaintext for all demo users, documented under "DEMO ONLY" in scripts/README.md.
- `backend/src/seed/factories/cases.factory.ts` — produces ~35 cases per tenant. Mix of statuses (OPEN / IN_PROGRESS / RESOLVED), priorities (LOW / MEDIUM / HIGH / CRITICAL), and ages (today through 90 days back). Each case assignedTo a deterministic engineer in its tenant.
- `backend/src/seed/util/deterministic-uuid.ts` — `seedUuid(parts: string[]): string` — namespace UUID v5 from `(seedVersion, ...parts)`. Pure function, no I/O.
- `backend/src/seed/util/seeded-prng.ts` — seeded pseudo-random generator for non-UUID determinism (e.g., picking which engineer a case is assigned to). xorshift32 or similar — small, fast, no dependency.
- `backend/src/seed/seed.cli.ts` — entrypoint for `yarn db:seed:demo`. Creates a standalone Nest application context (no HTTP server, just DI), invokes `SeedService.seedDemo()`, exits.
- `backend/src/seed/constants.ts` — `SEED_VERSION` literal + canonical tenant slugs + the known demo password plaintext. Bumping `SEED_VERSION` invalidates ALL prior seed UUIDs and re-seeds from scratch.

### Backend — modified files

- `backend/src/app.module.ts` — register `SeedModule` (only loaded when invoked by the CLI; never wired into the HTTP runtime path).
- `backend/package.json` — add `"db:seed:demo": "ts-node src/seed/seed.cli.ts"` script.

### Scripts — modified files

- `scripts/dev.sh` — three new subcommands:
  - `demo:seed [--target=local|rds]` — runs the seed against local DB by default; with `--target=rds` connects to the deployed RDS (see Decision B below for the path).
  - `demo:reset [--target=local|rds]` — alias for `demo:seed` (idempotent by design — re-running IS the reset).
  - `demo:credentials` — prints the canonical demo email/password pairs in a formatted table.
- `scripts/README.md` — "DEMO ONLY" section documenting demo credentials + the `--target=rds` confirmation prompt + the `NODE_ENV=production` refusal.

## Demo dataset shape (concrete)

Three tenants, identical structure per tenant. All values below derive deterministically from `SEED_VERSION = 1` plus the tenant slug.

### Tenants

| Slug | Name |
|---|---|
| `abc-company` | ABC Company (existing default — seed adopts it rather than recreate) |
| `acme-services` | Acme Services |
| `globex-support` | Globex Support |

### Users per tenant (10 each, 30 total)

| Role | Count | Status | Notes |
|---|---|---|---|
| ADMIN | 1 | ACTIVE | One admin per tenant for the approval-flow demo |
| ENGINEER | 3 | ACTIVE | Three engineers per tenant so case assignment varies |
| AGENT | 2 | ACTIVE | Two agents per tenant |
| CUSTOMER | 4 | ACTIVE | Three active + one PENDING_APPROVAL (4 rows total) |

Email pattern: `{role}{n}@{tenant_slug}.demo` — e.g., `engineer1@acme-services.demo`, `admin1@abc-company.demo`. All passwords: `demo123!` (bcrypt-hashed during seed, plaintext documented).

### Cases per tenant (~35 each, ~105 total)

Distributed across:
- **Status:** ~10 OPEN, ~15 IN_PROGRESS, ~10 RESOLVED
- **Priority:** ~3 CRITICAL, ~10 HIGH, ~15 MEDIUM, ~7 LOW
- **Age:** ~5 from today, ~10 within 7d, ~10 within 30d, ~10 within 90d
- **Assignment:** evenly distributed across the 3 engineers per tenant

Each case has a realistic title + description drawn from a fixed pool of 35 case archetypes (login issue, billing question, integration failure, etc.). The pool is hand-curated — no Faker — so screenshots stay accurate run to run.

## Decisions to confirm before code lands (3 architectural calls)

Per Pre-flight 25, three sub-decisions remained open. I have recommendations for each; this is the moment to lock them in.

### Decision A — Where the seed code lives

**Option 1 (recommended) — `backend/src/seed/` as a NestJS module.** Pros: imports entities directly, uses `TypeOrmModule.forFeature` and `@InjectRepository` patterns that already exist elsewhere in the backend, gets free TypeScript checking against the entity schema, runs in the same DI container the app uses. Cons: requires a CLI entrypoint script that bootstraps a standalone Nest app.

**Option 2 — `scripts/seed-demo.ts` as a standalone TS script.** Pros: no Nest bootstrap dance. Cons: must hand-construct a TypeORM data source, reimplement the entity registration, lose the existing patterns.

**Recommendation:** Option 1. The Nest bootstrap cost is ~15 lines in the CLI entrypoint; the upside is the seed code reads like the rest of the backend.

### Decision B — How the RDS target is reached

**Option 1 (recommended for P1) — Direct connection from local machine** using the `db:promote-rds` pattern we already have for the first-admin bootstrap. Script temporarily flips RDS to `publicly_accessible=true`, runs the seed against `<rds-endpoint>:5432`, then trap-cleans `publicly_accessible=false` on exit (success or failure).

**Option 2 — ECS one-shot task.** A pre-built backend image runs the seed as a task in the same VPC as RDS; no public RDS toggle needed. Safer for prod, more plumbing to set up. Right pattern for P12 prod hardening.

**Option 3 — Admin API endpoint.** `POST /admin/seed-demo` behind an admin guard. Operationally simple, expands the prod attack surface even when the endpoint is gated.

**Recommendation:** Option 1. The pattern is already proven (Mid-flight 8 ish — the `db:promote-rds` flow). For P12 prod readiness we'll move to Option 2; until then, Option 1 is cheap and the surface is your own dev account.

### Decision C — Idempotency marker

**Option 1 — `seed_origin` column on every seedable entity.** Explicit, but requires schema changes to four entities. Intrusive.

**Option 2 — Tenant-slug prefix convention.** All seeded data lives in tenants whose slug starts with `demo-` (or is in a known list). Reset truncates by slug. Cons: requires renaming `abc-company` to `demo-abc-company` or accepting that the default tenant is "demoable."

**Option 3 (recommended) — Deterministic UUIDs as their own marker.** Since `seedUuid()` produces the same UUID every run from the same inputs, the seeder knows exactly which rows it created. Reset path: enumerate every UUID the seeder WOULD create at this version, `DELETE` by ID, then insert fresh. No schema changes, no slug surgery, idempotent by construction.

**Recommendation:** Option 3. Smallest blast radius. The same hashing logic that makes the seed deterministic IS the idempotency mechanism — no new concept, no schema impact, no naming convention to remember.

## Acceptance criteria

- [ ] `yarn db:seed:demo` against a fresh local DB produces 3 tenants, 30 users, ~105 cases. Re-running produces no errors and leaves the dataset identical (same UUIDs, same counts).
- [ ] `scripts/dev.sh demo:seed` with no `--target` flag = local DB. With `--target=rds` prompts for confirmation, then runs against the deployed RDS using the temp-public-access pattern (and cleans up on exit).
- [ ] `scripts/dev.sh demo:reset` is equivalent to `demo:seed` (idempotent — calling either name produces the same end state).
- [ ] `scripts/dev.sh demo:credentials` prints a table of demo emails + the shared `demo123!` password, with a clearly-marked "DEMO ONLY" header.
- [ ] Seed refuses to run when `NODE_ENV=production` (process-level guard at the top of `seed.cli.ts`).
- [ ] After running seed, log in as `admin1@abc-company.demo` / `demo123!` — Dashboard renders with cases visible in MyOpenCases (admin's role gets engineer's case load by deterministic assignment), Recent Activity shows recent updates.
- [ ] Log in as a user from `acme-services` and confirm in the network tab that `GET /cases` returns ONLY `acme-services`-tenanted cases (multi-tenant isolation proof).
- [ ] Bumping `SEED_VERSION` from 1 → 2 and re-running invalidates all old IDs, deletes the old rows, creates new ones. Verified locally.
- [ ] `yarn workspace @caseflow-ai/backend lint && typecheck && test` clean.

## Resolved pre-flight decisions

- **Pre-flight 25 — Demo seed harness.** Established the scope, dataset shape, idempotency requirement, and entry points. This spec confirms the implementation details.

## Out of scope (deferred)

- **Data for entities that don't exist yet** — customer organizations (P2), notes (P3), attachments (P4), AI artifacts (P8). The seed module is structured so adding new entity factories is additive.
- **Faker-style random data.** Hand-curated pool stays deterministic.
- **A web UI to trigger seeding.** Script-only. P12 might add an admin-only endpoint behind heavy gating.
- **Demo-data isolation from real user data.** Any rows you create as a real user during demos persist until the next reset (they're untouched by the seeder because their UUIDs aren't in the seed's known-ID list). This is intentional — lets you "use the demo" without losing your work.

## How we'll work this sub-phase

Same chunked pattern as P1.7 and P1.8.

### Chunk A — Backend seed module + dataset
1. `backend/src/seed/` directory with factories, util, service, module, CLI entrypoint.
2. `package.json` script registration.
3. Local-DB run produces the dataset cleanly.
4. Unit tests for `seedUuid()` (determinism), `seededPrng()` (reproducibility), and the seed orchestrator (idempotency — running twice doesn't duplicate).

**Review checkpoint 1.** Show the user:
- The seed module structure.
- A snippet of `seedUuid()` and a demonstration that two runs produce identical IDs.
- Output of running `yarn db:seed:demo` twice and a screenshot showing the same row count both times.

### Chunk B — `dev.sh` wrappers + RDS target path
1. `demo:seed` / `demo:reset` / `demo:credentials` subcommands in `scripts/dev.sh`.
2. `--target=rds` flow with the temp-public-access pattern (reuse the existing `db:promote-rds` helper if possible).
3. `scripts/README.md` "DEMO ONLY" section with the credentials table.

**Review checkpoint 2.** User runs `dev.sh demo:seed` against local + `demo:credentials` to see the output. RDS-target path is reviewed but NOT executed yet — that's saved for the final checkpoint.

### Chunk C — Seed the deployed RDS + final verification
1. `dev.sh demo:seed --target=rds` against the live `caseflow.musto.io` backend's RDS.
2. Browser verification: log in as `admin1@abc-company.demo`, confirm Dashboard renders with realistic data.
3. Multi-tenant proof: log in as a user from a different tenant, confirm `GET /cases` returns disjoint data.

**Review checkpoint 3.** User drives the deployed demo end-to-end. After approval: commit + push, mark P1.9 ✅, **Phase 1 closes at 9/9.**

## Risks

- **`SEED_VERSION` bump invalidates prior seed data.** Anyone running an older `dev.sh demo:reset` against a v2 seed environment would create duplicate rows. Mitigation: include the `SEED_VERSION` in the seeder's output so the user always knows which version they ran.
- **RDS temp-public-access window.** During `demo:seed --target=rds`, the RDS instance is briefly publicly accessible. The trap cleanup makes this window ~10-30 seconds, but a determined attacker scanning for new public Postgres instances could in theory hit the window. Mitigation: this is the dev environment, the password is in Secrets Manager (not on the public internet), and the window is brief. For prod (P12) we move to Option B from Decision B (ECS one-shot task).
- **Demo user `demo123!` password is visible in the repo.** Acceptable for dev — demo users are clearly fenced (`.demo` email TLD, dataset wiped on next seed). The seed refuses to run with `NODE_ENV=production`; if anyone ever tried to seed prod, the guard catches it.

## Followup

After P1.9 ships:
- **Phase 1 closes at 9/9** — the first complete phase in the project. ROADMAP updates: P1 ✅, "Last shipped" rewritten, all P1 acceptance criteria boxes checked.
- **P2 (Customer model) extends the seed** — adds customer organizations + contacts to the dataset. Pattern is additive; no refactor of P1.9 work.
- **Content series Article 02 (decision logs) launches the same week** — the deployed demo is now a live receipt to point readers at.

### Public demo access — register-and-approve, NOT published credentials

The seeded demo users (`admin1@abc-company.demo`, etc.) are for **internal use only** — they populate the dataset so a new viewer sees real content immediately, and they're available for demos Armando runs himself. They are **NOT** published publicly.

Public demo access works through the **register-and-approve** flow instead. The LinkedIn audience that arrives at caseflow.musto.io is invited to:

1. Visit the site and register a real account (real name, real email)
2. DM Armando on LinkedIn that they've registered
3. Armando promotes their status to ACTIVE via the existing admin approval endpoint
4. They log in, see the seeded dataset through their own user, can poke around freely

Why this beats published credentials:

- **You know who's reviewing.** Every demo visitor self-identifies. Pre-qualified soft-lead funnel for recruiters and engineering leaders.
- **The approval workflow IS the demo.** P1.2's PENDING_APPROVAL → ACTIVE state machine is one of the more interesting architectural pieces in the project — letting a viewer experience the gate firsthand demonstrates the pattern better than any blog post.
- **No shared credentials in the wild.** Published demo credentials get crawled, brute-forced, end up in someone's dotfiles.
- **Per-user sandboxing.** Each viewer's own user means their experimentation doesn't step on anyone else's view.

What lands in `publishing/` (the standalone leadership/culture surface) is the **invitation language**, not the credentials. Something like:

> Want to see CaseFlow AI for yourself? Register at https://caseflow.musto.io, then DM me with the email you used — I'll approve your account and you can drive the engineer dashboard with the same demo data the screenshots show.

That gets added to the project README + the long-form Article 01 + Article 02 announcements when they ship.

### Post-P1 close-out tasks (separate from P1.9 implementation)

Once P1.9 ships and P1 is 9/9, the close-out sequence is:

1. **Merge `phase/p1-multi-tenancy` into `main`.** Merge-commit (not squash) — preserves sub-phase commit history for the decision-log-style archaeology pattern we've been maintaining.
2. **Tag `v0.1.0` on `main`** — semantic-versioning anchor for the first complete-phase milestone. Release notes link to the decision log + ROADMAP + deployed URL.
3. **Add screenshots to the root `README.md`** (still on the private repo) to promote the project to anyone with access:
   - Engineer Dashboard with seeded data (`MyOpenCases` + `RecentActivity` showing realistic content)
   - The tour engine in action (beacon visible on first login)
   - The ROADMAP (snapshot of the per-phase progress table) — proves the project is alive, not abandoned
   - Decision log (screenshot of a real Pre-flight + Mid-flight entry pair) — proves the methodology is in practice, not theoretical
4. **Stand up the public `caseflow-ai-showcase` repo.** The main repo stays private. The showcase repo is the curated portfolio surface:
   - `README.md` — hero copy + screenshots + "try the demo" CTA (register-and-approve flow)
   - `ROADMAP.md` — kept in sync at phase close-outs
   - `docs/decisions/` — the full decision log (copied from the main repo at close-out time)
   - `docs/specs/` — the dev-ready task specs (P1.1 through P1.9)
   - `docs/architecture/` — overview diagrams + the AI-Ready SDLC SVG from Article 01
   - `docs/patterns/` — **curated code excerpts** illustrating specific patterns (10–20 line snippets with context, NOT whole files):
     - Widget registry contract from `features/dashboard/types.ts`
     - Role gate in `TourEngine.tsx`
     - RTK `condition` pattern from `casesSlice.ts`
     - Decision log Mid-flight 9 / 10 / 11 entries as illustrations of the format
     - Terragrunt module-vs-live-stack separation
   - `docs/articles/` — the long-form articles as they publish (cross-reference to publishing/)
   - `LICENSE` — clear license for the curated excerpts
5. **Publish the "register + DM me" demo invitation** — invitation language goes in the showcase repo's README, the long-form Article 01, and future article launch posts. Public demo access is via self-identification, not shared credentials.
6. **Update the publishing registry** — mark `P1-foundation-complete` as a published milestone with the v0.1.0 tag URL and the showcase repo URL.

### Why the showcase pattern

Standing up a curated public surface (rather than making the main repo fully public) accomplishes three things:

- **The methodology spotlight stays bright.** Decision logs, specs, ROADMAP — fully indexed, sharable, the unique differentiator. No noise from "you should have used X" debates over implementation choices.
- **Production-parity feel.** Senior production codebases are private; the showcase repo with rich methodology artifacts mirrors how real work happens at real orgs. That mirror IS the credibility signal.
- **Reversible decision.** Main repo stays private; showcase repo is curated at phase close-outs. If feedback later suggests the full code would help, flipping the main repo public has zero cost. The reverse (un-publishing) is impossible.

Effort cost: ~2 hours to stand up the first version, then ~30 minutes per phase close-out to keep it in sync. The cadence already maps to phase close-outs — no separate maintenance burden.

### Optional bonus — "Why I kept the repo private" article

This decision IS an article in its own right. Contrarian take that lines up with the target audience (engineering leaders) and the existing thesis (judgment over throughput, methodology over code). Worth slotting in as a leadership-and-culture standalone piece — possibly Article 12.5 in the series, possibly its own standalone. Captured as an idea in publishing/registry.md to revisit when the showcase repo lands.

These tasks aren't P1.9 deliverables strictly — they're the formal close of Phase 1. They get their own commits + their own ROADMAP "Last shipped" line once they happen.
