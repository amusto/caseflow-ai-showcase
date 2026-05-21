# Dev-Ready Task — P1.1 · Tenant entity + TypeORM migrations bootstrap

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.1**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Land the `Tenant` entity and the TypeORM migrations pipeline so every
subsequent P1 sub-phase has a tenant-shaped schema to attach to. Add
`tenantId` to `Case` and `CaseHistory`. **No behavioral changes to the
cases service** in this sub-phase — tenant scoping arrives in P1.5.

## Business / user value

Multi-tenant SaaS is the product direction. P1.1 is the schema-shaped
foundation: without it, every sub-phase that follows would either
hard-code single-tenant behavior (and need a rewrite) or block on data
modeling. Shipping migrations now (instead of leaning on
`synchronize: true`) makes the dev schema diff explicit, which catches
silent entity drift early.

## Constraints

- **Architecture:** Pre-flight decision 1 — `JWT_SECRET` from env via
  `ConfigService`. Pre-flight 2 — JWT-claim-only tenant resolution
  (relevant in P1.4, but `Tenant` entity shape supports it). Pre-flight 4
  — `synchronize: false`, migrations are schema authority.
- **Security:** New env var `JWT_SECRET` documented in `.env.example`
  with a non-secret placeholder. Real value is per-engineer in
  uncommitted `.env`. `Tenant.slug` is URL-safe and unique-indexed (it
  has to be safe for future use in URLs even if subdomain resolution is
  deferred per pre-flight 2).
- **Data:** Pre-flight 3 — dev DB is wiped (`yarn infra:reset`) before
  migrations run. `tenantId` is `NOT NULL` from migration #1.
- **Delivery:** Conventional Commit: `feat(backend): introduce Tenant entity and migrations bootstrap (P1.1)`.

## Affected areas

- **Controllers:** none (no behavioral change yet).
- **Services:** none in this sub-phase.
- **DTOs:** none in this sub-phase.
- **Entities:**
  - New: `backend/src/tenants/entities/tenant.entity.ts`
  - Modified: `backend/src/cases/entities/case.entity.ts` (adds
    `tenantId` column + `@ManyToOne Tenant`)
  - Modified: `backend/src/cases/entities/case-history.entity.ts`
    (adds `tenantId` column + `@ManyToOne Tenant`)
- **Module:** New `backend/src/tenants/tenants.module.ts` registering
  the Tenant repository. Imported by `AppModule`.
- **Shared config:** New `backend/src/database.config.ts` exporting a
  `databaseConfig: TypeOrmModuleOptions` const. `AppModule`'s
  `forRootAsync` spreads it with `autoLoadEntities: true`.
- **Migrations: deferred to P10.** Per Mid-flight 1 reversal, dev uses
  `synchronize: true`. The originally-written initial migration is
  preserved at `docs/reference/migrations/` as a reference artifact.
  No `backend/src/data-source.ts`, no `backend/src/migrations/` dir,
  no `migration:*` scripts in `backend/package.json` pre-P10.
- **Env:** `backend/.env` + `backend/.env.example` add
  `JWT_SECRET=changeme-dev-only`. **Known config-load-order issue**
  with the static-const `databaseConfig` shape — see Mid-flight 3 in
  the decision log; fix deferred until a non-default env value is
  actually needed.
- **Tests:**
  - `backend/src/tenants/tenants.service.spec.ts` — even though there's
    no service logic yet, write the unit test for `Tenant` entity shape
    (column presence, slug uniqueness assertion via Jest schema check).
  - Defer cases-service-with-tenant-scope tests to P1.5 per decision log.
- **Docs:** No new docs in this sub-phase; ROADMAP "Phases at a glance"
  P1 row updates the "Where to look" cell after merge.

## Acceptance criteria

- [ ] `yarn infra:reset && yarn infra:up && yarn dev:backend` brings the schema up cleanly via `synchronize: true` (per Mid-flight 1 reversal). The `tenants` table and the `tenant_id` columns on `cases` + `case_history` are visible in Adminer at `localhost:8081`.
- [ ] `backend/src/database.config.ts` has `synchronize: true` with a comment pointing at the decision log; `backend/src/app.module.ts` does NOT set `migrationsRun`.
- [ ] `yarn workspace @caseflow-ai/backend typecheck` clean.
- [ ] `yarn workspace @caseflow-ai/backend lint` clean.
- [ ] `yarn workspace @caseflow-ai/backend test` clean (existing `cases.service.spec.ts` must still pass — no behavioral changes expected).
- [ ] `JWT_SECRET` documented in `backend/.env.example` (placeholder value, not a real secret).
- [ ] Conventional Commit message lands on `phase/p1-multi-tenancy` branch.

## Test plan

- **Unit tests:**
  - Tenant entity column shape (`name`, `slug` are present; `slug` unique).
  - Existing cases service spec unchanged — proves no behavioral regression.
- **Integration / e2e tests:** none in this sub-phase (no behavioral surface).
- **Manual verification:**
  - `yarn infra:reset && yarn infra:up`
  - `yarn workspace @caseflow-ai/backend migration:run`
  - Open Adminer (`http://localhost:8081`), confirm `tenants`, `cases.tenant_id` (NOT NULL + FK), `case_history.tenant_id` (NOT NULL + FK).
  - `yarn dev:backend` boots without errors.

## Security / privacy notes

- `JWT_SECRET` is added in this sub-phase but **not yet used** — P1.2
  introduces JWT issuance. Adding the env var here keeps the
  `.env.example` diff out of the auth-shaped PR and avoids the "where
  did this env var come from" review confusion.
- `Tenant.slug` is unique-indexed; collisions become 409 in P1.4 when
  tenant creation surfaces over HTTP.
- No new PII is added.

## Rollback plan

- The migrations directory is brand new — `yarn migration:revert` undoes
  in reverse order. Three reverts get back to pre-P1.1 schema.
- If the dev DB is in a weird state, `yarn infra:reset` is the nuclear
  option and is the documented dev workflow per pre-flight 3.
- Code-level rollback: revert the merge commit on `phase/p1-multi-tenancy`.

## Out of scope (deferred to later sub-phases)

- `User` entity, password hashing, basic-auth login → **P1.2**
- Google OAuth + JIT → **P1.3**
- Replacing `MockUserGuard` with `JwtAuthGuard` + tenant context → **P1.4**
- Scoping cases queries by `tenantId` → **P1.5**
- Frontend login → **P1.6**

## How we'll work this sub-phase

Spec-first per `docs/ai-sdlc.md`. Driver TBD — set at start of the
coding block. Default mode: Armando reviews; agent drives the
migration scaffolding because TypeORM CLI wiring is mechanical and
easy to get exactly right with a clear spec.
