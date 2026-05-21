# Dev-Ready Task — P1.4 · JwtAuthGuard + ActiveUserGuard (global) + MockUserGuard refactor

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.4**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Close the data-access gate. Every protected endpoint now requires a
valid access token AND an ACTIVE user status. `MockUserGuard` moves
behind a default-off env flag so dev can still iterate fast without
going through real auth. P1.3 (Google OAuth + JIT) is unaffected — it
becomes another way to mint a JWT that this guard chain happens to
accept.

## Business / user value

Without P1.4, the security promise of P1.2 is incomplete:

- PENDING users get a JWT but no API enforces the gate. They can read
  case data today by passing the JWT they got from `/auth/login`.
- Cases endpoints currently trust `MockUserGuard` headers — any
  unauthenticated request with `x-user-role: ADMIN` is treated as
  admin. That's fine for solo dev; catastrophic in any shared environment.

P1.4 makes the auth layer actually enforce what P1.2's data model promises.

## Constraints

- **Architecture:** Global `APP_GUARD` providers (Pre-flight 10). Public
  endpoints opt out via `@Public()` decorator.
- **Security:** `JwtAuthGuard` rejects expired / malformed / wrong-secret
  tokens. `ActiveUserGuard` rejects non-ACTIVE users with structured
  error codes (`ACCOUNT_PENDING_APPROVAL`, `ACCOUNT_DISABLED`).
- **Backward compat for dev:** `MockUserGuard` env-gated per Pre-flight 11.
- **Delivery:** Conventional Commit:
  `feat(backend): JwtAuthGuard + ActiveUserGuard (P1.4) + MockUserGuard env-gated`.

## Affected areas

**Decorators (new):**
- `backend/src/auth/decorators/public.decorator.ts` — `@Public()` sets
  reflector metadata read by both auth guards.
- `backend/src/auth/decorators/current-user.decorator.ts` — `@CurrentUser()`
  parameter decorator returns `request.user` (typed as `RequestUser`).

**Guards (new):**
- `backend/src/auth/guards/jwt-auth.guard.ts` — Reflector-aware:
  short-circuits `@Public()` endpoints, otherwise validates the Bearer
  token via `JwtService.verify`, attaches `{ id, tenantId, role, status }`
  to `request.user`. Throws `UnauthorizedException` on missing/invalid token.
- `backend/src/auth/guards/active-user.guard.ts` — runs after JwtAuthGuard.
  Short-circuits `@Public()`. Otherwise asserts `request.user.status ===
  ACTIVE`; throws `ForbiddenException` with `code: 'ACCOUNT_PENDING_APPROVAL'`
  or `'ACCOUNT_DISABLED'` for the other states.

**Mock guard refactor:**
- `backend/src/common/guards/mock-user.guard.ts` — `RequestUser` type
  expanded to `{ id, tenantId, role, status }`. Reads `x-user-id`,
  `x-user-role`, `x-tenant-id`, `x-user-status` headers with safe
  defaults. Respects `@Public()` (returns `true` early). Never blocks
  by itself — it only populates `request.user`.

**App wiring:**
- `backend/src/app.module.ts` registers two `APP_GUARD` providers in
  order: (a) `JwtAuthGuard` OR `MockUserGuard` (per env flag), then
  (b) `ActiveUserGuard`. Both guard classes added to the module's
  `providers` list so the DI container can construct them.

**Controllers:**
- `backend/src/auth/auth.controller.ts` — each endpoint gets `@Public()`.
- `backend/src/cases/cases.controller.ts` — drops
  `@UseGuards(MockUserGuard)`. Uses `@CurrentUser() user: RequestUser`
  instead of `@Request() req`.
- `backend/src/users/users.controller.ts` — same refactor.

**Service plumbing:**
- `backend/src/cases/cases.service.ts` — `RequestUser` type now includes
  `tenantId` + `status`. Service methods stay otherwise identical
  (tenant-scoped queries land in P1.5, not here).

**Tests:**
- `backend/src/auth/guards/jwt-auth.guard.spec.ts` — public bypass, valid
  token attaches user, missing token → 401, expired token → 401,
  wrong-signature token → 401.
- `backend/src/auth/guards/active-user.guard.spec.ts` — public bypass,
  ACTIVE → pass, PENDING_APPROVAL → 403 with `ACCOUNT_PENDING_APPROVAL`,
  DISABLED → 403 with `ACCOUNT_DISABLED`.
- `backend/src/cases/cases.service.spec.ts` — update RequestUser shape
  in existing mocked users.

**Env:**
- `backend/.env` + `backend/.env.example` add:
  `USE_MOCK_USER_GUARD=false` with a "Dev only — never enable in prod"
  comment.

## Acceptance criteria

- [ ] `curl GET /cases` with no Authorization header → 401.
- [ ] `curl GET /cases` with a valid ACTIVE-user JWT → 200 (returns that user's cases per existing P1.2 behavior; tenant-scoping is P1.5).
- [ ] `curl GET /cases` with a valid PENDING_APPROVAL JWT → 403 with `{ "code": "ACCOUNT_PENDING_APPROVAL" }`.
- [ ] `curl GET /cases` with a valid DISABLED JWT → 403 with `{ "code": "ACCOUNT_DISABLED" }`.
- [ ] `curl POST /auth/login` without a token → still works (endpoint is `@Public()`).
- [ ] With `USE_MOCK_USER_GUARD=true`, `curl GET /cases -H 'x-user-role: CUSTOMER' -H 'x-user-status: ACTIVE'` → 200, no JWT required.
- [ ] With `USE_MOCK_USER_GUARD=true` and `x-user-status: PENDING_APPROVAL` → 403 (ActiveUserGuard still enforces).
- [ ] `PATCH /users/:id/status` with `x-user-role: CUSTOMER` (mock) or non-admin JWT → 403.
- [ ] `yarn lint && yarn typecheck && yarn test` clean.

## Test plan

- **Unit:** see guard specs above; mock `Reflector` + `JwtService`.
- **Manual:** smoke-test the four states (no token / ACTIVE / PENDING / DISABLED) against `/cases`. Toggle `USE_MOCK_USER_GUARD` and confirm both modes work.

## Security / privacy notes

- JWT secret read via `ConfigService.get('JWT_SECRET')` — same load-order safety as `databaseConfig` (Mid-flight 4 pattern).
- Failed verification messages are intentionally generic ("Invalid token", "Account inactive") — no enumeration.
- `USE_MOCK_USER_GUARD` flag must NEVER be set in production. Documented in `.env.example` with a loud warning; P10 hardening adds a startup assertion that fails the boot when `NODE_ENV=production && USE_MOCK_USER_GUARD=true`.

## Rollback plan

- Single commit on `phase/p1-multi-tenancy`. Revert with `git revert <sha>`.
- If guards misfire and lock everyone out: set `USE_MOCK_USER_GUARD=true` in `.env`, restart backend. Mock takes over and you're back to header-based identity.

## Out of scope (deferred)

- Tenant-scoped cases queries → **P1.5**
- Frontend login + gated-page routing → **P1.6**
- API key auth for service-to-service → future hardening
- Production assertion against `USE_MOCK_USER_GUARD=true` → P12 hardening

## How we'll work this sub-phase

Per Option 1: agent drives, user reviews diffs.
