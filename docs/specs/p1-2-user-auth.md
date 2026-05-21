# Dev-Ready Task — P1.2 · User entity + basic auth

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.2**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Ship real authentication: a `User` entity with bcrypt-hashed passwords,
`POST /auth/register`, `POST /auth/login`, both returning a signed JWT
that carries `sub` (user id), `tenantId`, and `role` claims. No changes
to the cases service yet — `MockUserGuard` still in play; replacing it
with `JwtAuthGuard` is P1.4.

## Business / user value

P1.1 created the tenant boundary in the schema. P1.2 makes that boundary
addressable — there are now real users who belong to tenants and can
prove identity. Everything from P1.5 onward (tenant-scoped queries,
audit logs, AI invocations) needs a real `userId` + `tenantId` at the
request boundary. This is also the canonical assessment-shaped slice:
controller + service + DTO + bcrypt + JWT + class-validator + tests.

## Constraints

- **Architecture:** AuthModule depends on UsersModule (lookup by email);
  UsersModule depends on TenantsModule (tenant existence check on
  register). No circular imports.
- **Security:** Passwords stored only as bcrypt hashes; never logged,
  never returned in responses. JWT signed with `process.env.JWT_SECRET`
  via NestJS `JwtModule.registerAsync` (so config-load-order doesn't
  bite us again — Mid-flight 4 pattern).
- **DTOs:** `class-validator` decorators on every field. `email` →
  `@IsEmail()`, `password` → `@IsString() @MinLength(8)`. Invalid
  payloads return 400 via the global `ValidationPipe` already in `main.ts`.
- **No bypass paths:** `MockUserGuard` is unchanged in this sub-phase
  but `/auth/*` endpoints are explicitly public (`@Public()` decorator
  or `@UseGuards()` omission). The replacement happens in P1.4.
- **Delivery:** Conventional Commit: `feat(backend): real auth via bcrypt + JWT (P1.2)`.

## Affected areas

- **Entities:**
  - New: `backend/src/users/entities/user.entity.ts` (UUID PK, email
    unique-indexed per tenant, password hash, role enum, tenantId FK,
    timestamps).
- **Modules:**
  - New: `backend/src/users/users.module.ts` (registers User repo,
    exports for AuthModule).
  - New: `backend/src/auth/auth.module.ts` (registers AuthService +
    AuthController; imports UsersModule + JwtModule).
- **Services:**
  - New: `backend/src/users/users.service.ts` — `findByEmail`,
    `createUser` (with bcrypt hashing).
  - New: `backend/src/auth/auth.service.ts` — `register(dto)`,
    `login(dto)`, `signToken(user)`. Validation, password compare,
    JWT issuance live here; controllers stay thin.
- **Controllers:**
  - New: `backend/src/auth/auth.controller.ts` — `POST /auth/register`,
    `POST /auth/login`. Decorated with `@Public()` (or omit the global
    guard for /auth/* paths via main.ts setup — TBD).
- **DTOs:**
  - New: `backend/src/auth/dto/register.dto.ts` —
    `email`, `password`, `name`, `tenantId`. All validated.
  - New: `backend/src/auth/dto/login.dto.ts` — `email`, `password`.
- **Tests:**
  - New: `backend/src/auth/auth.service.spec.ts` — register happy
    path, register duplicate-email rejection, login happy path, login
    wrong-password rejection, login unknown-email rejection.
- **App wiring:**
  - `backend/src/app.module.ts` imports `UsersModule` and `AuthModule`.
- **Env:**
  - `JWT_SECRET` already shipped in P1.1. P1.2 adds `JWT_EXPIRES_IN`
    (default `24h`) and `BCRYPT_COST` (default `12`) to
    `backend/.env` + `backend/.env.example`.
- **Dependencies added (justified):**
  - `@nestjs/jwt` — NestJS-idiomatic JWT module, integrates with
    `ConfigService` and `Reflector`.
  - `bcrypt` — battle-tested password hashing. Alternative
    (`argon2`) is stronger but bcrypt is what most assessment
    codebases use; consistency wins.
  - `@types/bcrypt` (dev).
  - Passport NOT added in P1.2 — `JwtAuthGuard` (with passport-jwt)
    arrives in P1.4 when MockUserGuard is replaced. Keeping P1.2
    focused.

## Acceptance criteria

- [ ] `curl -X POST http://localhost:3000/auth/register -H 'Content-Type: application/json' -d '{...}'` creates a user and returns a JWT.
- [ ] Re-registering the same email returns 409 (conflict).
- [ ] `curl -X POST http://localhost:3000/auth/login` with valid credentials returns the same JWT shape.
- [ ] Login with wrong password returns 401, **identical response shape to "unknown email"** (no email enumeration).
- [ ] Decoding the JWT shows `sub`, `tenantId`, `role`, `iat`, `exp`. Token expires per `JWT_EXPIRES_IN`.
- [ ] DTO validation: missing `email` → 400; weak password → 400; `tenantId` not a UUID → 400.
- [ ] Register fails if `tenantId` doesn't refer to an existing tenant (404 or 400 — TBD per pre-flight).
- [ ] Password is never logged or returned in any response payload.
- [ ] `yarn workspace @caseflow-ai/backend lint && typecheck && test` clean.

## Test plan

- **Unit (`auth.service.spec.ts`):**
  - Register: happy path, duplicate email → 409, missing tenant → error
  - Login: happy path, wrong password → 401, unknown email → 401 (same shape as wrong password), inactive user (future: skip for now)
  - `signToken`: claims include sub/tenantId/role; expiry matches config
- **Integration:** Defer to a future test-harness sub-phase; mocked-repo unit tests cover P1.2.
- **Manual verification:**
  - Seed a tenant: directly insert one row via Adminer or a small `yarn seed-tenant` one-off.
  - `curl` register → verify token decodes correctly at jwt.io.
  - `curl` login → verify same token shape.

## Security / privacy notes

- Password is hashed at `BCRYPT_COST` (default 12 → ~250ms per hash; intentional friction against brute force).
- JWT is signed, not encrypted — claims are visible to anyone holding the token. **Do not put PII in claims.** Just `sub`, `tenantId`, `role`, plus standard JWT claims (`iat`, `exp`).
- `JWT_SECRET` lives in env. Production (P10) moves it to Secrets Manager.
- Login response is identical for "unknown email" vs "wrong password" — prevents user enumeration.
- DTOs use `class-validator`'s `whitelist: true` (already configured globally in `main.ts`) so unexpected fields are stripped — protects against mass-assignment attacks.

## Rollback plan

- Single commit on `phase/p1-multi-tenancy`. Revert with `git revert <sha>`.
- Schema rollback: with `synchronize: true`, deleting the `User` entity file and restarting the backend will drop the `users` table on next boot (or you can manually `DROP TABLE users` in Adminer). The schema follows the entities.

## Out of scope (deferred to later sub-phases)

- Google OAuth + JIT provisioning → **P1.3**
- `JwtAuthGuard` replacing `MockUserGuard` on cases endpoints → **P1.4**
- Tenant-scoped cases queries using `request.user.tenantId` → **P1.5**
- Frontend login UI → **P1.6**
- Refresh token strategy → deferred (likely production-hardening, P12)
- Password reset flow → deferred (out of P1)
- Email verification → deferred (out of P1)
- Admin-invite-user flow → deferred (out of P1)

## How we'll work this sub-phase

Per the chosen mode: agent drives implementation; user reviews diffs.
Three pre-flight decisions need resolution before code starts —
captured below for the decision log entry.

## Resolved pre-flight decisions

Captured in `docs/decisions/phase-01-multi-tenancy-auth.md`:

- **Pre-flight 5 — JWT strategy.** Short access token (15 min) +
  opaque refresh token (7 days) hashed in `refresh_tokens` table, rotated
  on every `/auth/refresh` use.
- **Pre-flight 6 — Password policy.** `@MinLength(8)`, no composition rules.
- **Pre-flight 7 — Registration scope + email uniqueness.** Open public
  self-registration with `tenantId` in body; email is **globally unique**
  across tenants.
- **Pre-flight 8 — Admin approval gate.** New users land in
  `status: PENDING_APPROVAL`. They **cannot log in** until an admin
  flips them to `ACTIVE` via `PATCH /users/:id/status`. The admin
  endpoint uses `MockUserGuard` + manual role check until P1.4.

## Affected areas (revised after decisions)

**Entities:**
- New: `backend/src/users/entities/user.entity.ts` — UUID PK, `email`
  (globally unique-indexed), `passwordHash`, `name`, `role` (enum from
  `common/enums/user-role.enum.ts`), `status` (new enum), `tenantId` FK,
  timestamps.
- New: `backend/src/common/enums/user-status.enum.ts` —
  `PENDING_APPROVAL | ACTIVE | DISABLED`.
- New: `backend/src/auth/entities/refresh-token.entity.ts` — UUID PK,
  `userId` FK, `tokenHash` (sha256 of the opaque token), `expiresAt`,
  `revokedAt` (nullable), `createdAt`.

**Modules:**
- New: `backend/src/users/users.module.ts` — registers User repo +
  UsersService + UsersController; exports UsersService for AuthModule.
- New: `backend/src/auth/auth.module.ts` — registers AuthService +
  AuthController; imports UsersModule + JwtModule.registerAsync (reads
  `JWT_SECRET` via ConfigService — same load-order safety as
  databaseConfig per Mid-flight 4).

**Services:**
- New: `backend/src/users/users.service.ts` — `findByEmail`,
  `findById`, `createUser` (bcrypt hashes inside; sets status =
  PENDING_APPROVAL), `updateStatus`, `assertExists`.
- New: `backend/src/auth/auth.service.ts` — `register(dto)`,
  `login(dto)`, `refresh(token)`, `logout(token)`, `issueTokenPair(user)`,
  internal `verifyRefreshToken`, `revokeRefreshToken`.

**Controllers:**
- New: `backend/src/auth/auth.controller.ts` —
  `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`,
  `POST /auth/logout`. Unauthenticated (no guard).
- New: `backend/src/users/users.controller.ts` —
  `PATCH /users/:id/status` (admin-only via MockUserGuard + manual
  role check). No other CRUD here; that's deferred.

**DTOs:**
- New: `backend/src/auth/dto/register.dto.ts` —
  `email` (`@IsEmail`), `password` (`@IsString @MinLength(8)`),
  `name` (`@IsString @MinLength(1) @MaxLength(80)`),
  `tenantId` (`@IsUUID`).
- New: `backend/src/auth/dto/login.dto.ts` — `email` + `password`.
- New: `backend/src/auth/dto/refresh.dto.ts` — `refreshToken` (`@IsString`).
- New: `backend/src/users/dto/update-user-status.dto.ts` —
  `status` (`@IsEnum(UserStatus)`, restricted to `ACTIVE | DISABLED`
  in validation — admins can't downgrade to PENDING).

**Tests:**
- New: `backend/src/auth/auth.service.spec.ts` —
  register happy, register duplicate email → 409, login pending → 403,
  login disabled → 403, login wrong password → 401 (same shape as
  unknown email), refresh happy + rotates token, refresh revoked-token → 401,
  logout revokes.
- New: `backend/src/users/users.service.spec.ts` —
  createUser stores hash (not plain), updateStatus persists.

**App wiring:**
- `backend/src/app.module.ts` imports `UsersModule` and `AuthModule`.

**Env (additions):**
- `JWT_EXPIRES_IN=15m`
- `REFRESH_TOKEN_EXPIRES_IN=7d`
- `BCRYPT_COST=12`
- `JWT_SECRET` is already shipped from P1.1.

**Dependencies added (justified):**
- `@nestjs/jwt` — Nest-idiomatic JWT module.
- `bcrypt` + `@types/bcrypt` — password hashing.
- `crypto` is built-in (no install) — used for the opaque refresh
  token + SHA256 hash.

## Acceptance criteria (revised)

- [ ] `curl -X POST /auth/register` creates a user in `PENDING_APPROVAL`
  and **returns 201 with a message + the user record (no tokens)**.
  Tokens are issued only after activation.
- [ ] Duplicate email → 409.
- [ ] `curl -X POST /auth/login` for a `PENDING_APPROVAL` user → 403
  with `code: 'ACCOUNT_PENDING_APPROVAL'`.
- [ ] After `PATCH /users/:id/status { "status": "ACTIVE" }` (sent
  with `x-user-role: ADMIN` per MockUserGuard), the same login returns
  200 with `{ accessToken, refreshToken, expiresIn }`.
- [ ] Wrong password and unknown email both return 401 with **identical
  body shape** (no enumeration).
- [ ] `POST /auth/refresh { "refreshToken": "..." }` returns a new
  token pair; the old refresh token is revoked (next use returns 401).
- [ ] `POST /auth/logout` revokes the supplied refresh token.
- [ ] JWT decodes to `{ sub, tenantId, role, iat, exp }`; expiry matches
  `JWT_EXPIRES_IN`.
- [ ] DTO validation: missing email → 400, weak password → 400, bad
  tenantId UUID → 400.
- [ ] `PATCH /users/:id/status` without ADMIN role → 403.
- [ ] Password is never logged or returned.
- [ ] `yarn workspace @caseflow-ai/backend lint && typecheck && test` clean.

## Bootstrap recipe (first admin)

Pre-flight 8 creates a chicken-and-egg: no one can log in until an
admin approves them, and there are no admins. To bootstrap a tenant:

```bash
# 1. Insert a tenant (via Adminer or psql):
INSERT INTO tenants (id, name, slug) VALUES (gen_random_uuid(), 'Demo Tenant', 'demo');

# 2. POST /auth/register to create a user against that tenant's id.
# 3. Promote that user via Adminer:
UPDATE users SET status='ACTIVE', role='ADMIN' WHERE email='you@example.com';

# 4. That user is now the bootstrap admin. Future users can be
#    approved via PATCH /users/:id/status sent with x-user-role: ADMIN.
```

A `yarn seed-admin` script lives in the [`docs/roadmap.md`](../../../../docs/roadmap.md)
backlog and likely lands during P1.6 or as a P1 closing convenience.

## Test plan (revised)

- **Unit (`auth.service.spec.ts`):** see Tests list above.
- **Unit (`users.service.spec.ts`):** createUser writes a hash not the
  plain password; updateStatus persists.
- **Manual verification:** register → expect 201 + PENDING → attempt
  login → expect 403 → promote via DB → login → expect 200 + tokens →
  refresh → expect new pair → use old refresh → expect 401.

## Out of scope (deferred to later sub-phases)

- Google OAuth + JIT provisioning → **P1.3**
- `JwtAuthGuard` replacing `MockUserGuard` on cases endpoints (and on
  `/users/:id/status` itself) → **P1.4**
- Tenant-scoped cases queries → **P1.5**
- Frontend login UI → **P1.6**
- Password reset, email verification, MFA → deferred (out of P1)
- Per-user permission overrides beyond role → deferred (out of P1)

## How we'll work this sub-phase

Per Option 1: agent drives implementation, user reviews diffs. After
implementation, commit on `phase/p1-multi-tenancy` with
`feat(backend): real auth via bcrypt + JWT with refresh + admin approval (P1.2)`.
