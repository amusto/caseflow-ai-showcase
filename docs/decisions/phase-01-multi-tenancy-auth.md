# Phase 1 — Multi-tenancy + Real Auth · Decision Log

**Phase:** [P1](../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-)
**Opened:** 2026-05-17
**Status:** Open — sub-phases in progress.

This log captures the irreversible-shaped decisions made at the start of
Phase 1, before any code lands. Each decision states what was chosen, why
the alternative was rejected, and what would trigger revisiting it. New
mid-phase decisions get appended in the order they happen.

---

## Pre-flight 1 — JWT secret source for dev

**Decision.** `JWT_SECRET` is sourced from `process.env.JWT_SECRET` for
both dev and prod in Phase 1. Backend reads it through
`ConfigService.get<string>('JWT_SECRET')` so swapping the source later
doesn't touch consumers. The dev value is documented in
`backend/.env.example` with a non-secret placeholder; real prod secrets
land in AWS Secrets Manager in **P10**.

**Alternative rejected.** Introduce a small `AppSecrets` provider
abstraction in P1 that wraps env access today and Secrets Manager later.
Rejected because it adds a layer with one implementation, which is
premature — `ConfigService` already gives us the indirection we need.
The refactor in P10 is ~30 minutes of mechanical work; the
"future-proofing" version would cost the same now plus add
explain-this-class friction in every PR review of P1.

**Revisit if.** A non-Lambda runtime adds direct-AWS-SDK Secrets Manager
fetching in the request path (then a wrapper helps); or P10 ships before
P1.6 (the frontend isn't usable without auth, so the order matters).

---

## Pre-flight 2 — Tenant resolution

**Decision.** Backend resolves tenant **from the JWT claim only**. The
`tenantId` claim is set at token issuance (login + Google OAuth JIT).
`JwtAuthGuard` populates `request.user` with `{ id, tenantId, role }`. A
NestJS interceptor (or guard) extracts `tenantId` into a request-scoped
`TenantContext` provider that the services inject when scoping queries.

**Alternative rejected.** JWT claim **plus** subdomain fallback
(`acme.caseflow.app`). Rejected because subdomain resolution adds DNS,
wildcard ACM cert, and a proxy-layer tenant injector — none of which
exist in P10 yet, and all of which expand the attack surface for token
spoofing. Multi-tenant SaaS does not require subdomains; subdomains are
a branding decision.

**Revisit if.** A specific customer demands a branded subdomain, or
when P12 prod hardening would benefit from subdomain-level WAF rules.

---

## Pre-flight 3 — Existing dev-database cases without `tenant_id`

**Decision.** **Wipe and re-seed.** The P1.1 migration adds the
`Tenant` table and the `tenant_id` column to `cases` + `case_history`
as `NOT NULL` from the start. The existing dev DB is reset before the
migration runs:

```bash
yarn infra:reset && yarn infra:up   # drops the volume, restarts fresh
# (TypeORM synchronize: false will be set; migrations create the schema)
```

There is no production data to preserve.

**Alternative rejected.** Add `tenant_id` nullable in P1.1, backfill all
existing rows to a `legacy` tenant in P1.5, then flip `NOT NULL`. This
rehearses the production-shape pattern — but caseflow-ai has no
production DB yet, and the rehearsal value is captured separately in
P10's RDS migration sub-phase. Doing the pattern here just to do it
adds two migrations and a backfill script for zero current value.

**Revisit if.** Anyone gets a long-lived shared dev DB (e.g., a hosted
dev RDS that multiple engineers share before P10 lands).

---

## Pre-flight 4 — TypeORM schema management mode

**Decision (implied by Pre-flight 3, surfaced for clarity).** Phase 1
turns off `synchronize: true` and adopts TypeORM migrations as the
schema authority. `backend/src/data-source.ts` becomes the CLI entry
point for migration generation/runs. Scripts added to
`backend/package.json`: `migration:generate`, `migration:run`,
`migration:revert`, `migration:show`. `AppModule`'s
`TypeOrmModule.forRootAsync` sets `synchronize: false` for all envs and
`migrationsRun: true` for non-test envs so the backend self-applies
pending migrations on boot.

**Alternative rejected.** Keep `synchronize: true` for dev convenience
and adopt migrations only when P10 deploys to a real DB. Rejected
because `synchronize: true` masks the actual schema diff at every
entity change — engineers stop noticing when a column they thought they
added wasn't. The cost of running migrations in dev is one extra
`yarn migration:run` per pull; the benefit is parity with prod from
day one.

**Revisit if.** Migration friction in dev becomes the bottleneck — at
which point a `db:reset` script (drop + migrate + seed) covers the
common case without going back to `synchronize: true`.

---

## Pre-flight 5 — JWT lifetime + refresh strategy (P1.2)

**Date:** 2026-05-17 (P1.2 opening).
**Decision.** Short-lived access token (15 min) signed with `JWT_SECRET`
+ long-lived opaque refresh token (7 days) stored hashed in a
`refresh_tokens` table. Refresh endpoint rotates the refresh token on
every use — the old one is revoked, a new one is issued alongside the
new access token. Logout revokes the refresh token.

**Alternative rejected.** Single 24h token, no refresh — simpler but
ships a known weakness: any leaked token is valid for a full day with
no revocation path. Refresh-token rotation is the production-grade
pattern; doing it now means P1 ships the production auth shape and
P12 hardening doesn't need to retrofit the schema.

**Why opaque (not JWT) refresh tokens.** Opaque random strings hashed
in the DB cannot be tampered with offline; revocation is a single
update in the `refresh_tokens` row; no second JWT secret to manage.
JWT refresh tokens would force us to track JWT IDs (`jti`) anyway —
that's just an opaque token with extra steps.

**Revisit if.** Mobile clients require longer offline windows
(refresh expiry might extend); or auth deployment moves behind a
gateway that handles tokens externally (Cognito, Auth0).

---

## Pre-flight 6 — Password minimum length

**Decision.** 8 characters minimum via `@MinLength(8)` on the
`RegisterDto`. No complexity rules (no required digits / uppercase /
special chars).

**Why no complexity rules.** Modern OWASP guidance (and NIST SP 800-63B)
prefers length over composition. Length-only rules are also what most
assessment codebases validate against — consistency with the broader
ecosystem matters for portfolio readability.

**Revisit if.** A specific customer compliance regime (HIPAA, SOC 2
audit findings, government contracts) requires composition rules; or
P11 observability data shows brute-force attempts that length alone
isn't repelling.

---

## Pre-flight 7 — `/auth/register` scope + email uniqueness

**Decision (scope).** Open public self-registration. `POST /auth/register`
accepts `{ email, password, name, tenantId }` — anyone can register
against any existing tenant. The endpoint is unauthenticated
(`@Public()` decorator or explicit guard omission).

**Decision (email uniqueness).** Email is **globally unique** across
all tenants (single index on `users.email`). A given person cannot
have two accounts in the system. Trade-off: simpler login (email +
password is sufficient — no tenant disambiguation needed); cost: a
person cannot legitimately belong to two tenants under the same email.

**Alternative rejected.** Email unique-per-tenant via composite index
on `(tenant_id, email)`. More flexible for multi-tenant-per-user
scenarios, but forces login to ask "which tenant?" or to use a
subdomain (pre-flight 2 explicitly deferred subdomains). Not worth
the UX cost for v1.

**Why public registration in a SaaS context.** Open registration is
unusual for B2B SaaS but a deliberate dev convenience here. In
production posture (P10+), this either (a) gets locked behind an
admin-invite flow, (b) restricted to specific tenant slugs, or (c)
gated by email-domain allowlist. Pre-flight 8 adds an admin-approval
gate that mitigates the immediate risk: anyone can register, but no
one can actually log in until an admin approves them.

**Revisit if.** P10 introduces a customer-facing client app
(roadmap P14 stretch) — that app's signup flow needs a different
shape (magic link, email verification, possibly tenant auto-create).

---

## Pre-flight 8 — Account activation gated by admin approval

**Date:** 2026-05-17 (mid-P1.2 spec, added as a hard requirement).
**Decision (revised same day — see "Amendment" below).** Newly-registered
users land in `status: PENDING_APPROVAL`. They **can log in** to a
gated UI surface that tells them they're awaiting approval, but
they **cannot access tenant data** until an admin updates their
status to `ACTIVE`.

**Implementation shape (revised).**

- `UserStatus` enum: `PENDING_APPROVAL | ACTIVE | DISABLED`.
- `User.status` column, default `PENDING_APPROVAL` (registration sets
  this; no escape hatch).
- `AuthService.login`:
  - DISABLED → 403 (`code: 'ACCOUNT_DISABLED'`).
  - PENDING_APPROVAL or ACTIVE → issues access + refresh tokens.
  - JWT payload includes `status` claim: `{ sub, tenantId, role, status, iat, exp }`.
  - Login does not leak whether the email exists (wrong password and
    unknown email both return the same 401 shape).
- **Data-access gate moves to P1.4.** When `JwtAuthGuard` replaces
  `MockUserGuard`, a sibling `ActiveUserGuard` (or an
  `@RequireActive()` decorator) rejects PENDING_APPROVAL tokens on
  any endpoint that touches tenant data. Pure-self endpoints (profile,
  logout, refresh) stay open to PENDING users.
- Frontend (P1.6) reads the `status` claim and routes
  PENDING_APPROVAL users to a gated "awaiting approval" page rather
  than the case list. ACTIVE users get the normal UI.
- `PATCH /users/:id/status` admin endpoint with body
  `{ status: 'ACTIVE' | 'DISABLED' }`. Guarded by `MockUserGuard`
  with a manual role check until P1.4 swaps in `JwtAuthGuard`.
- Bootstrap: still requires one admin promoted via Adminer for the
  first tenant. See `users/dto/update-user-status.dto.ts` + the P1.2
  task spec's "Bootstrap recipe" section.

**Amendment (same day, before any code shipped).** Original Pre-flight 8
blocked PENDING_APPROVAL users at login. Revised to allow login + use
the JWT `status` claim to gate data access at the API layer and route
to a gated page at the UI layer. **Why the change:** the original
shape forced users into a confusing "your credentials are valid but
you can't get a token" state with no path forward — they'd see a 403
and assume the registration didn't work. The revised shape gives
them a clear "you're in the system, an admin is reviewing" message,
which is the standard SaaS UX.

**Why this matters at all.** Open self-registration (Pre-flight 7) is
otherwise a bare-API security hazard. The status gate makes
self-registration safe: anyone can create an account, but no one can
read or modify tenant data until a human approves them.

**Revisit if.** A magic-link email-verification flow lands (P14
customer client) — that flow can auto-promote to ACTIVE on first
email click, bypassing manual admin approval for self-service tenants.

---

## Pre-flight 9 — Default tenant for self-registration

**Date:** 2026-05-17 (mid-P1.2 spec, added alongside the Pre-flight 8
amendment).
**Decision.** `POST /auth/register` does NOT take a `tenantId` in the
request body. New users are auto-attached to a single default tenant
seeded by the backend on application bootstrap. Default tenant:
`name = "ABC Company"`, `slug = "abc-company"`. Slug + name overridable
via `DEFAULT_TENANT_SLUG` and `DEFAULT_TENANT_NAME` env vars; defaults
match what's documented above.

**Implementation shape.**

- New `TenantsService.ensureDefault()` — idempotent: looks up by slug,
  creates if missing. Returns the `Tenant` row.
- `TenantsModule` implements `OnModuleInit` and calls
  `ensureDefault()` on boot.
- `UsersService.createUser()` no longer accepts `tenantId`; it calls
  `TenantsService.findDefault()` (cached read after boot) and attaches
  the new user to that tenant.
- `RegisterDto` drops the `tenantId` field.

**Alternative rejected.** Pass `tenantId` in the request body (the
original Pre-flight 7 shape). Rejected because:
1. Front-end UX awkward — what would the registration form even ask
   for? "Paste a tenant UUID"? Subdomain-based tenant resolution
   (Pre-flight 2 deferred) would solve this, but it's not in scope.
2. Demo-friendly default tenant means a new visitor can register and
   land in a working state immediately. No "create a tenant first"
   prerequisite.
3. B2B "invite to your company's tenant" flow is a future capability
   (likely a future P1.x or the P14 customer client). Defaulting now
   doesn't preclude that — we add an `invitedToTenantId` parameter
   later and treat its absence as "use the default."

**Why "ABC Company".** Reviewer-friendly placeholder — same role as
"Acme Co." or "Lorem Ipsum Inc." Communicates "this is the demo
default" without claiming brand. Trivially overridable via
`DEFAULT_TENANT_NAME` for a real deployment.

**Revisit if.**
- A real multi-tenant onboarding flow lands (invite-to-tenant,
  signup-creates-tenant, subdomain-based tenant resolution per
  deferred Pre-flight 2).
- The default-tenant pattern becomes a security risk because
  attackers exploit "default tenant has lots of legitimate data
  alongside test users." Unlikely pre-P10, but worth noting.

---



---

## Pre-flight 10 — Guard application: global with @Public() opt-out (P1.4)

**Date:** 2026-05-17 (P1.4 opening).
**Decision.** Register `JwtAuthGuard` and `ActiveUserGuard` as global
`APP_GUARD` providers in `AppModule`. Every endpoint is protected by
default. Public endpoints (`/auth/register`, `/auth/login`,
`/auth/refresh`, `/auth/logout`, plus a future `/health`) opt out via
a `@Public()` decorator that sets reflector metadata the guards check
first.

**Alternative rejected.** Per-controller `@UseGuards(JwtAuthGuard,
ActiveUserGuard)` on every protected controller. More explicit (the
guard chain is visible on each controller header), but creates a
"forgot to add the guard on the new controller" failure mode that
APP_GUARD eliminates by default-deny. Production-grade default is
default-deny.

**Why this matters now.** P1.5 introduces tenant-scoped cases queries
that **must** be guarded. Without a global guard, P1.5 would need to
remember to add `@UseGuards` on every new endpoint — a process invariant
we'd rather enforce structurally.

**Revisit if.** A future endpoint type (webhook receivers, public APIs
with their own auth like API keys) needs a fundamentally different guard
chain — at which point we add a third guard or accept per-controller
overrides for those specific cases.

---

## Pre-flight 11 — MockUserGuard kept behind `USE_MOCK_USER_GUARD` env flag

**Date:** 2026-05-17 (P1.4 opening).
**Decision.** `MockUserGuard` stays in the codebase but only activates
when `process.env.USE_MOCK_USER_GUARD === 'true'`. When the flag is on,
`MockUserGuard` registers as the first `APP_GUARD` *instead of*
`JwtAuthGuard` — it reads identity from headers (`x-user-id`,
`x-user-role`, `x-tenant-id`, `x-user-status`) with safe defaults
(`role: CUSTOMER`, `status: ACTIVE`). When the flag is off (default),
`JwtAuthGuard` is the APP_GUARD and the mock is unreachable.

`ActiveUserGuard` runs in both modes. Mock requests with
`x-user-status: PENDING_APPROVAL` will be rejected at the data-access
gate exactly like real PENDING JWT requests — letting devs rehearse the
gated-UX flow without going through registration.

**Alternative rejected.** Fully delete `MockUserGuard`. Cleaner, but
costs every smoke test the login round-trip. Env-flag opt-in keeps the
clean path the default while preserving the fast-iteration option.

**Implementation shape.**

- `MockUserGuard.canActivate` reads:
  - `x-user-id` → `request.user.id` (default `'demo-user-id'`)
  - `x-user-role` → `request.user.role` (default `UserRole.CUSTOMER`)
  - `x-tenant-id` → `request.user.tenantId` (default `'demo-tenant-id'`)
  - `x-user-status` → `request.user.status` (default `UserStatus.ACTIVE`)
- `AppModule` selects between `JwtAuthGuard` and `MockUserGuard` at boot:
  ```ts
  {
    provide: APP_GUARD,
    useClass: process.env.USE_MOCK_USER_GUARD === 'true'
      ? MockUserGuard
      : JwtAuthGuard,
  }
  ```
- `.env.example` documents the flag with a `# Dev convenience only — DO NOT enable in prod` warning.

**Revisit if.** A real load-testing or fuzz-testing tool needs
identity-spoofing without going through JWT issuance. In that case
the test rig sets the flag in its own env, dev unchanged.

**Removal trigger.** When P10 cloud infra goes live with real RDS +
Secrets Manager, the flag is removed entirely. The grace period exists
only for pre-deployment dev iteration.

---

## Pre-flight 12 — Cross-tenant access returns 404, not 403 (P1.5)

**Date:** 2026-05-17 (P1.5 opening).
**Decision.** When a Tenant A user requests a case by ID that belongs
to Tenant B, the API returns `404 Not Found` — indistinguishable from
"this case doesn't exist at all." Cross-tenant access does not surface
as `403 Forbidden` or any other status that would confirm the case
exists in another tenant.

**Why 404 (not 403).** B2B SaaS multi-tenancy depends on tenant
boundaries being unobservable from outside the tenant. A 403 response
leaks information ("this case exists, you just can't see it"), which
lets an attacker enumerate case IDs across tenants. 404 collapses two
failure modes — "doesn't exist" and "belongs to another tenant" — into
one indistinguishable response.

**Implementation shape.** `CasesService.findOne` adds
`tenantId: user.tenantId` to its `where` clause. TypeORM's `findOne`
returns `null` when no row matches. The existing
`if (!caseEntity) throw new NotFoundException(...)` line already
produces 404 — no new code path needed for the cross-tenant case. The
same flows through `update` (which calls `findOne` internally).

**Trade-off accepted.** Slightly worse DX during legitimate
misconfigurations (an engineer testing across tenants sees 404, not
"oh I'm in the wrong tenant"). Mitigated by: (a) the server logs
include `tenantId` mismatches at debug level for actual debugging;
(b) Adminer access lets dev verify schema state out-of-band.

**Revisit if.**

- A specific compliance regime (HIPAA audit trails, etc.) requires
  explicit "access denied" responses for forensic purposes.
- A customer support workflow needs to know "you tried to access
  another tenant's case" rather than "no such case."

In either case the right answer is probably a separate "support
operator" path that bypasses the 404 collapse, not a global change to
the customer-facing API.

---

## Pre-flight 13 — Vite proxy for same-origin cookie posture (P1.6)

**Date:** 2026-05-17 (P1.6 opening).
**Decision.** The Vite dev server proxies `/auth`, `/cases`, and
`/users` to `http://localhost:3000` (the NestJS backend). From the
browser's perspective, all API calls go to `localhost:8080` (the Vite
origin) — same-origin with the page. This sidesteps every CORS
complication that the httpOnly-cookie refresh-token transport
(Mid-flight 6 implemented in P1.6) would otherwise inflict on a
cross-origin setup.

**Why this matters.** SameSite=Strict cookies, credentialed CORS,
preflight OPTIONS, ACAO headers — none of that needs configuration when
the browser thinks the API is same-origin. Production (P10+) uses
CloudFront's `/api/*` cache behavior to do the same proxy at the edge,
so the production cookie posture matches dev byte-for-byte.

**Alternative rejected.** Full cross-origin setup with
`app.enableCors({ credentials: true, origin: 'http://localhost:8080' })`
on the backend, `withCredentials: true` on the frontend, and
`SameSite=None; Secure=true` on the cookie. Works (Chrome treats
localhost as a secure context), but adds three layers of config that
have to stay in sync across dev / staging / prod environments. The
proxy approach removes all three.

**Revisit if.** A standalone backend deployment (no edge proxy)
needs to be hit directly by a different-origin frontend. At that
point, enable credentialed CORS in addition to the proxy and pin
`SameSite=None; Secure` on the cookie.

---

## Pre-flight 14 — Status-based routing (P1.6)

**Date:** 2026-05-17.
**Decision.** Three route gates based on the access token's `status`
claim, implemented as React route wrappers:

- `<PublicOnly>` — `/login`, `/register`. Authenticated user redirects
  to `/` (if ACTIVE) or `/awaiting-approval` (if PENDING).
- `<RequireAuth>` — `/awaiting-approval`. Unauth → `/login`;
  ACTIVE → `/`.
- `<RequireActiveAuth>` — `/` and all main app routes. Unauth →
  `/login`; PENDING → `/awaiting-approval`; DISABLED → `/login`
  with error.

DISABLED users get the same redirect as unauthenticated (back to
login) — there's no "you're disabled" page because there's nothing
they can do from inside the app. The error message at `/login`
explains.

**Revisit if.** A new status value lands (e.g., `REQUIRES_2FA`,
`PASSWORD_EXPIRED`). Add a route + wrapper pair per status.

---

## Pre-flight 15 — POC deploy target: us-east-1, default account, `personal` AWS CLI profile

**Date:** 2026-05-17 (P10 opening — first deploy wave).
**Decision.** First deploy goes to your default AWS account, region
`us-east-1`, via the AWS CLI profile named `personal`. Both `live/dev`
and `live/prod` terragrunt configs pin these. `expected_account_id`
in `live/dev/terragrunt.hcl` and `live/prod/terragrunt.hcl` must be
filled in by you before the first apply — the generated provider's
`null_resource.account_guard` refuses to plan against the wrong
account.

**Why us-east-1.** CloudFront-fronting ACM certificates have to live
in us-east-1 regardless of where the rest of the stack runs. RDS
pricing is cheapest there. The default VPC has reasonable subnet
distribution.

**Revisit if.** Multi-region (probably never for a POC) or a specific
data-residency requirement.

---

## Pre-flight 16 — CloudFront default URL (no custom domain) for first deploy

**Date:** 2026-05-17.
**Decision.** Ship with CloudFront's default `*.cloudfront.net` URL.
No ACM certificate, no Route 53 record, no domain alias. Custom
domain becomes a follow-up sub-phase if needed.

**Why defer.** ACM in us-east-1 + Route 53 record + CloudFront alias
+ DNS propagation is its own ~30-minute path. None of it blocks the
"is this thing actually running on AWS" demo. We add it later when
there's a reason (real interview demo, recruiter link, etc.).

**Revisit if.** Recruiter or hiring manager URL needs to be branded;
or sharing a link in interview prep where the random CloudFront
hostname undermines polish.

---

## Pre-flight 17 — Database + JWT secrets via AWS Secrets Manager

**Date:** 2026-05-17.
**Decision.** Two AWS Secrets Manager secrets in `us-east-1`:

1. `caseflow-dev-db-password` — random password generated by Terraform's
   `random_password` resource, captured into Secrets Manager, fed into
   the RDS instance + the ECS task definition via the `secrets[]` block.
2. `caseflow-dev-jwt-secret` — random 64-byte hex generated by Terraform,
   fed into the ECS task as the `JWT_SECRET` env var.

Cost: ~$0.80/month ($0.40 per secret). Acceptable for the
production-shape posture this gives us.

**Alternative rejected.** SSM Parameter Store SecureString — free, same
KMS encryption story, slightly more friction to rotate. Picked
Secrets Manager because the rotation story is built-in and the
narrative ("secrets in Secrets Manager, IAM-scoped GetSecretValue on
the task execution role") is the production-grade answer for an
interview.

**Revisit if.** Cost becomes a real factor (it won't for a POC), or
secret rotation requirements appear.

---

## Pre-flight 18 — Manual `docker push` for first image; CI/CD deferred

**Date:** 2026-05-17.
**Decision.** First image to ECR goes via manual `docker buildx build`
+ `docker push` from the developer machine. No GitHub Actions
workflow, no OIDC role, no `.github/workflows/deploy-backend.yml` —
yet.

**Why defer.** The OIDC + Actions pipeline adds ~1 hour of agent work
(IAM provider, role, role policy, Actions workflow, ECR repo policy)
that doesn't unblock the first deploy. We add it once the first
deploy works end-to-end and the painful "wait, what's my ECR URI
again?" muscle memory is built.

**Revisit when.** First deploy is green and you've manually pushed
twice. The third manual push is the trigger for automating it.

---

## Pre-flight 19 — Custom domain caseflow.musto.io (supersedes Pre-flight 16)

**Date:** 2026-05-18 (end of P10 first deploy).
**Decision.** Wire the CloudFront distribution to `caseflow.musto.io`.
The `frontend_cdn` module gains an optional `domain_name` input;
when set, the module provisions the us-east-1 ACM cert and waits for
validation (`aws_acm_certificate_validation`). All resources are
count-gated on `local.has_domain` so the module still works for
default-URL deployments.

**DNS is managed at Network Solutions, not Route 53.** musto.io lives
at the registrar. The module deliberately does NOT touch DNS.

**Cert path: reuse existing wildcard.** The hardvard-timeboxing project
already validated a `*.musto.io` wildcard ACM cert in this account
(`us-east-1`). Re-using it skips the cert-creation + DNS-validation
dance entirely. The module accepts `acm_certificate_arn` as an
optional input — when set, the module uses it directly; when null,
the module creates a new cert and waits for DNS validation
(slower, harder path).

**Two-CNAME flow if you don't have a wildcard:** the module creates
the cert as PENDING; user adds the validation CNAMEs from
`acm_validation_records` output at the registrar; ACM polls and
ISSUES; CloudFront update completes; user adds the final alias CNAME
(`caseflow.musto.io` → `cloudfront_alias_target` output) at the
registrar. Total time: ~15-25 minutes if DNS is fast, more if
registrar is slow.

**One-CNAME flow with the wildcard (current path):** paste the
wildcard ARN into `acm_certificate_arn`, run `terragrunt apply`
(CloudFront update only — no cert wait), then add ONE CNAME at the
registrar (`caseflow.musto.io` → `cloudfront_alias_target` output).

**Supersedes Pre-flight 16.** First deploy intentionally used the
CloudFront default `*.cloudfront.net` URL to ship fast. With the
stack now stable, the branded URL is the better demo / portfolio
asset and the manual DNS step is a one-time cost.

**Why ACM lives in us-east-1.** CloudFront edge SSL certificates
must be in us-east-1 regardless of where the rest of the stack runs.
The env config (Pre-flight 15) is already us-east-1, so no separate
provider alias is needed.

**Trade-offs.**

- First apply takes ~15–25 min total: most of it waiting on DNS
  propagation + CloudFront edge updates.
- DNS records are added manually, twice (validation + alias). If
  musto.io moves to Route 53 later, the module gains the automatic
  path via a follow-up pre-flight.
- Backend ALB is still HTTP-only (no ACM cert on the ALB itself).
  CloudFront → ALB still rides HTTP. End-to-end TLS requires a
  second cert at the ALB layer + an api.* subdomain. Defer to a
  future hardening sub-phase.

**Revisit if.** Multiple branded domains needed (api.musto.io,
admin.musto.io, etc.) — at that point split the cert pieces into
their own module rather than embedding in `frontend_cdn`. Or, when
musto.io migrates to Route 53, restore the Route 53 automation that
was in this module's first draft (visible in git history of
`frontend_cdn/main.tf`).

---

## Pre-flight 20 — Google sign-in links to existing email-password user (P1.3)

**Date:** 2026-05-19 (P1.3 opening).
**Decision.** When a Google sign-in's email matches an existing
password-registered user, **link** by stamping `googleSub` on the
existing user row. Same user account, both auth methods work
afterward. No duplicate user row created.

**Why link.** Email is globally unique (Pre-flight 7). Allowing a
Google sign-in with the same email as a password user without linking
would require either (a) violating the unique invariant, or (b)
rejecting the sign-in with "email already in use." Linking is the
friendliest of the three.

**Mitigations against takeover.** Backend only links when Google
reports `email_verified = true` (Pre-flight 21 handles the
unverified-email case). The risk model is: an attacker who controls
the victim's Google account could sign in as them — but if the
attacker already owns the Google account that the victim registered
with, they already own the victim's identity at every other Google-SSO
property too. Not a new attack surface.

**Revisit if.** Customer compliance requirement demands explicit
account-linking confirmation (e.g. "we noticed your Google email
matches an existing account, click to link"). At that point, add a
linking-confirmation flow without redesigning the data model.

---

## Pre-flight 21 — Google-registered users land in PENDING_APPROVAL

**Date:** 2026-05-19.
**Decision.** Users created via Google JIT provisioning go through the
same admin-approval gate as password-registered users
(`status = PENDING_APPROVAL`). Frontend gates them to
`/awaiting-approval` until an admin promotes via
`PATCH /users/:id/status`.

**Alternative rejected.** Auto-ACTIVE on the basis that Google verifies
the email. Rejected because:
1. Email-verified ≠ tenant-approved. A multi-tenant SaaS where admins
   gatekeep membership shouldn't auto-admit anyone who has a Google
   account.
2. Consistency with Pre-flight 8 (revised). Mixing two onboarding
   flows where one auto-activates and the other doesn't is harder to
   reason about than one consistent gate.

**Revisit if.** A self-service tenant type lands where signup creates
its own tenant — at that point, the new-tenant-creator auto-promotes
to ADMIN on tenant creation, but subsequent invitees still gate.

---

## Pre-flight 22 — Token verification via `google-auth-library`

**Date:** 2026-05-19.
**Decision.** Use `google-auth-library`'s
`OAuth2Client.verifyIdToken({ idToken, audience: GOOGLE_CLIENT_ID })`
for ID token verification. The library handles:

- JWKS fetch + cache (Google rotates signing keys; the library handles
  rotation transparently).
- Signature verification.
- Audience check (token must be minted for OUR client ID, not a
  different OAuth app).
- Issuer check (must be `accounts.google.com` or `https://accounts.google.com`).
- Expiry check.

**Alternative rejected.** Manual JWKS verification via `jose` or
`jsonwebtoken`. Functionally equivalent but reinvents 50+ lines of
code that `google-auth-library` already battle-tests. The library is
Google's reference implementation.

**Revisit if.** A second OIDC provider lands (Microsoft, GitHub) —
at that point, abstract behind a generic `OidcVerifier` interface
and have provider-specific implementations. Google verifier stays
unchanged behind that interface.

---

## Mid-flight 1 — Reversal of pre-flight 4 (synchronize)

**Date:** 2026-05-17 (same day as the original pre-flight, after P1.1
implementation landed).
**Decision.** `synchronize: true` for all pre-P10 development. The
migration file shipped in P1.1 is preserved as a reference artifact in
`backend/src/migrations/` with a clarifying header; it is *not* auto-run
by the app. `migrationsRun` is removed from `app.module.ts`. The
TypeORM CLI scripts in `package.json` and `data-source.ts` stay
available for ad-hoc migration generation against the dev DB if needed.

**Why the reversal.** Pre-flight 4 optimised for parity with the
eventual production schema-authority pattern. In practice, pre-P10
phases churn entity shape frequently — Tenant adds `slug`, then maybe
`status`, then User and Customer arrive, then notes, attachments,
audit log. Each entity change becomes a migration ceremony that adds
friction without catching anything real, because the dev DB is the
only DB. The cost of `synchronize: true` (silent entity drift) only
hurts when there's *another* environment to drift from — and there
isn't one until P10.

**What this changes elsewhere.**

- `backend/src/database.config.ts` — `synchronize: true`, comment
  pointing here.
- `backend/src/app.module.ts` — `migrationsRun` line removed,
  comment pointing here.
- `backend/src/migrations/1779235200000-InitialSchemaWithTenants.ts`
  — kept, header rewritten to call out reference status + manual-run
  recipe.
- ROADMAP P1 acceptance criteria unchanged in spirit; the bullet
  about "migrations bring the schema up cleanly" is now satisfied by
  the equivalent `synchronize`-driven boot.

**Revisit if (revert triggers).**

1. **A second environment appears before P10.** If anyone provisions a
   hosted dev RDS (or a staging DB) that multiple machines share,
   `synchronize: true` becomes dangerous — flip back to migrations.
2. **P10 cloud infra starts.** That phase's RDS stack expects
   `synchronize: false`; the saved migration becomes the production
   baseline.
3. **A destructive entity change.** If an entity rename or column-type
   change risks silent data loss in dev, run the change as a hand-written
   migration once and revert to `synchronize: true` after.

**What is NOT reversed.**

Pre-flights 1, 2, 3 stand. JWT secret stays env-var sourced. Tenant
resolution remains JWT-claim-only. Dev DB still wipe-and-reseed (the
new behavior here is just *how* the schema gets created on that fresh
DB — TypeORM synchronize instead of migration run).

---

## Mid-flight 2 — Pin TypeORM via root `resolutions`

**Date:** 2026-05-17 (same day, after first `yarn dev:backend` post-P1.1).
**Trigger.** TS2322 from `AppModule`'s `TypeOrmModule.forRootAsync` factory
return value. The error message was a wall of text, but the cue was
`Type 'EntitySchema<any>' is not assignable to type 'EntitySchema<any>'`
— same type, two file paths.

**Root cause.** Yarn classic workspaces resolved TypeORM 0.3.28 into
*two* places on disk: `caseflow-ai/node_modules/typeorm/` (hoisted) and
`caseflow-ai/backend/node_modules/typeorm/` (un-hoisted). Identical
versions, but TypeScript treats two copies of the same lib as distinct
types because nominal type identity is per file. `databaseConfig()`
returned a `DataSourceOptions` from copy A; `TypeOrmModule.forRootAsync`
expected a `TypeOrmModuleOptions` whose `entities` reference copy B.
The compiler refused to unify them.

**Decision.** Add `"resolutions": { "typeorm": "0.3.28" }` to root
`package.json`. Yarn 1.22.x honors `resolutions`; the next `yarn install`
collapses both copies to one hoisted install.

**Why the error appeared now.** Pre-P1.1, no file imported a TypeORM
type into the AppModule chain. The cases module imported `Repository`
via `@nestjs/typeorm` indirection, which masked the duplication.
`databaseConfig()` was the first direct typeorm-type import, which
surfaced the latent issue.

**Manual cleanup needed once.**

```bash
rm -rf backend/node_modules/typeorm
yarn install
```

**Revisit if.** Another lib develops the same dual-install pattern
(common candidates: `reflect-metadata`, `rxjs`, `class-validator`).
Add to `resolutions` as needed. The first-line debugging cue is always
`Type 'X' is not assignable to type 'X'` — same name, different paths
in the long error.

**Assessment-relevant lesson captured.** Logged in
`prep/codesignal-may-22.md` as a new pattern. The provided codebase
will likely be a yarn workspace too; the same trap can appear during
the assessment.

---

## Mid-flight 3 — Removed unused migration scaffolding + flagged config-load-order

**Date:** 2026-05-17 (same day as Mid-flights 1 + 2).
**Trigger.** After Mid-flight 1 flipped `synchronize: true`, the migration
scaffolding (`data-source.ts`, the `typeorm` + `migration:*` scripts in
`backend/package.json`, the `dotenv` dep, the `migrations/` directory)
became dead weight. Worse — `databaseConfig()`'s return type
(`DataSourceOptions` from `typeorm` directly, required by
`data-source.ts`) was the boundary that surfaced the dual-install
problem behind Mid-flight 2. Removing the migration scaffolding lets
`databaseConfig` switch to `TypeOrmModuleOptions` from `@nestjs/typeorm`,
which never crosses into a direct `typeorm` type import.

**Decision.** Pre-P10, no migration tooling lives in `backend/`. The
single source of schema authority is the entity files + `synchronize: true`.
The migration originally shipped in P1.1 moves to
`docs/reference/migrations/` as a documented artifact for the P10 day.

**Code changes.**

- Deleted `backend/src/data-source.ts`.
- Deleted `backend/src/migrations/` directory.
- Removed scripts from `backend/package.json`: `typeorm`,
  `migration:create`, `migration:generate`, `migration:run`,
  `migration:revert`, `migration:show`.
- Removed `dotenv` from `backend/package.json` dependencies.
- Moved `1779235200000-InitialSchemaWithTenants.ts` to
  `docs/reference/migrations/` with an updated header explaining its
  reference-only status, plus a `README.md` describing how to revive
  the migration story for P10.
- `databaseConfig` switched from a `function(): DataSourceOptions` to
  a `const: TypeOrmModuleOptions`. Docstring tightened.

**Latent issue accepted, not fixed.** `databaseConfig` is a static
`const`, evaluated at module import time — before `ConfigModule.forRoot()`
loads `.env`. In dev this works because the `??` fallbacks happen to
match `.env` after the dev port (5433) was set as the default. **No
non-default env value will be picked up until this is fixed.** Fix
options when it matters:

1. Convert `databaseConfig` to a function called inside
   `TypeOrmModule.forRootAsync`'s `useFactory` (env reads then happen
   at runtime, after `ConfigModule`).
2. Inject `ConfigService` into the factory and read everything via
   `config.get<…>(…)` (Nest-idiomatic, drops the shared helper).

**Revisit if (deferred-fix triggers).**

- A CI environment sets a non-default `DATABASE_*` value and the
  backend fails to read it.
- P10 cloud infra introduces a `DATABASE_HOST` that isn't `localhost`
  (it will).
- Any test setup needs to override the default DB connection.

At any of those points, do one of the two fixes above.

---

## Mid-flight 4 — Function-form `databaseConfig` (closes Mid-flight 3's latent bug)

**Date:** 2026-05-17 (same day as Mid-flights 1-3).
**Trigger.** Mid-flight 3 documented the static-const `databaseConfig`'s
config-load-order bug and deferred the fix. Before adding any code that
would actually depend on `.env`-overridden values (P1.2 auth pulls
`JWT_SECRET`; P10 RDS will pull `DATABASE_HOST`), the latent bug becomes
a real bug. Fixing now is cheap.

**Decision.** Convert `databaseConfig` from a `const` of type
`TypeOrmModuleOptions` to a function returning `TypeOrmModuleOptions`.
`AppModule`'s `useFactory` calls `databaseConfig()` instead of spreading
the const. The function body runs at `useFactory` invocation, which Nest
sequences **after** `ConfigModule.forRoot()` has loaded `.env`.

**Code changes.**

- `backend/src/database.config.ts` — `export const databaseConfig: TypeOrmModuleOptions = { … };`
  becomes `export const databaseConfig = (): TypeOrmModuleOptions => ({ … });`.
  Docstring's "Known latent issue" paragraph removed; new docstring
  explains why the function form is intentional.
- `backend/src/app.module.ts` — `...databaseConfig` becomes
  `...databaseConfig()`.
- Default port fallback in `databaseConfig` returned from `5433` (the
  workaround value) to `5432` (Postgres default) since the real value
  now correctly comes from `.env`.
- Verified locally: backend boots, connects to the dev DB on the port
  declared in `.env` (5433).

**Why not `ConfigService` injection.** Cleaner option B from the prep doc
would be `inject: [ConfigService]` + `config.get<string>('…')` in the
factory. Rejected because (a) it requires inlining the entire DB config
in `app.module.ts`, killing the shared `databaseConfig` helper, and
(b) the function-form is one keystroke per file and equally correct for
this use case. ConfigService can come back later if we grow more
complex env-shaped config (e.g. per-environment schema validation).

**Closes (no longer revisit-triggers from Mid-flight 3).**

- CI sets a non-default `DATABASE_*` value → now picked up.
- P10 cloud RDS endpoint → now picked up.
- Test setup overrides DB connection → now picked up.

**Assessment-relevant lesson preserved.** Pattern #9 in
`prep/codesignal-may-22.md` now reads "we initially accepted the
workaround, then applied the proper fix" with both forms shown — that's
the narrative the assessment is likely to test against.

---

## Mid-flight 6 — Auth transport: Bearer-in-body now, httpOnly cookie for refresh in P1.6

**Date:** 2026-05-17 (post P1.2 implementation, before frontend exists).
**Surfaced by.** Reviewing P1.2's `AuthController` after implementation
landed: `login` and `refresh` return `{ accessToken, refreshToken,
tokenType: 'Bearer', expiresIn }` in the JSON response body. The choice
of transport (Bearer-in-body vs httpOnly cookie vs hybrid) was implicit
in the code, never explicit in the decision log. Surfacing now.

**Current state (P1.2 ships).** Pure Bearer-in-body.

- `POST /auth/login` → tokens in JSON response.
- `POST /auth/refresh` → refresh token in JSON request body, new tokens
  in JSON response.
- `POST /auth/logout` → refresh token in JSON request body.
- No `Set-Cookie` headers, no cookie-parser middleware, no CORS
  credentialed mode.

**Decision.** Keep Bearer-in-body for P1.2. **Switch refresh token to
httpOnly cookie in P1.6** when the browser frontend lands. Access token
stays in the response body (kept in JS memory only, never `localStorage`).

**Why defer the switch.**

1. Pre-P1.6 there's no browser client. The XSS risk that httpOnly cookies
   mitigate doesn't apply to curl, Postman, or jest tests.
2. The switch adds cookie middleware, CSRF protection (or
   `SameSite=Strict`), credentialed CORS, and 4-ish test rewrites.
   That work has no value until there's a browser actually consuming
   the API.
3. P1.6 is the natural place to make the change — the frontend code
   needs to change in tandem (store access in Redux state, drop the
   refresh token from any client-side storage). Doing both atomically
   in one phase keeps the diff coherent.

**Why hybrid (not full cookie or full bearer) is the long-term target.**

- **Access token in body / memory:** simple to attach via `Authorization: Bearer <jwt>`
  header; expires fast (15 min) so XSS-stolen tokens have a short
  usable window; works for non-browser clients without rework.
- **Refresh token in httpOnly cookie:** unreadable by JS, scoped to
  `/auth/*`, `SameSite=Strict` + same-origin requirement defangs CSRF;
  the long-lived credential never lives anywhere JS can reach.

The all-cookie alternative forces every endpoint to be cookie-credentialed,
which makes mobile / server-to-server consumers awkward. The all-Bearer
alternative leaves the long-lived refresh token JS-readable, which is
the XSS hazard everyone warns about.

**Implementation diff for P1.6 (what we'll do then).**

```ts
// auth.controller.ts (sketch)
@Post('login')
async login(@Body() dto: LoginDto, @Res({ passthrough: true }) res: Response) {
  const pair = await this.auth.login(dto);
  res.cookie('refreshToken', pair.refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    path: '/auth',
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });
  const { refreshToken: _omit, ...rest } = pair;
  return rest; // { accessToken, tokenType, expiresIn }
}

@Post('refresh')
async refresh(@Req() req: Request, @Res({ passthrough: true }) res: Response) {
  const token = req.cookies?.refreshToken;
  if (!token) throw new UnauthorizedException();
  const pair = await this.auth.refresh(token);
  res.cookie('refreshToken', pair.refreshToken, { /* same options */ });
  const { refreshToken: _omit, ...rest } = pair;
  return rest;
}
```

Plus: `cookie-parser` middleware in `main.ts`, credentialed CORS allowlist,
`@types/cookie-parser`, an `Auth.smoke.spec.ts` update, and a curl
recipe update.

**Revisit triggers (move the switch earlier).**

- A browser-shaped client lands earlier than P1.6.
- A security review flags the localStorage-or-memory ambiguity.
- A mobile client demands different transport (kicks the question
  back open — possibly all-Bearer wins after all).

**What does NOT change in P1.2.**

The Bearer-in-body shape is correct and tested. No code change needed
right now. This entry exists so the choice is explicit and the switch
path is mapped.

---

## Mid-flight 7 — Aligned `@JoinColumn` names with `@Column` property names (refresh-token 500)

**Date:** 2026-05-17 (post P1.5 commit, surfaced by `scripts/dev.sh refresh`).
**Trigger.** Calling `POST /auth/refresh` returned `500 Internal Server
Error`. The user object inside `AuthService.refresh` was `undefined`,
which crashed `stored.user.status`. Root cause was the latent
JoinColumn naming bug I flagged in the P1.2 wrap-up and deferred.

**Root cause.** Across four entities I had `@Column` properties
(camelCase: `userId`, `tenantId`) paired with `@JoinColumn({ name: 'snake_case' })`:

```typescript
@Column({ type: 'uuid' })
userId!: string;                              // → DB column "userId"

@ManyToOne(() => User, { onDelete: 'CASCADE' })
@JoinColumn({ name: 'user_id' })              // → declares FK column "user_id"
user!: User;
```

Under `synchronize: true`, TypeORM created **both** columns:
- `userId` (scalar, populated by `repo.save({ userId: ... })`).
- `user_id` (FK target, always `NULL` because nothing assigned
  `entity.user`).

`findOne` with `relations: { user: true }` joins on `user_id` →
no match → relation comes back undefined → `.status` access throws → 500.

**Why it surfaced only on `/auth/refresh`.** That's the first
endpoint that *loads a relation*. Earlier P1.5 cases work only writes
scalar `tenantId` and queries by it — never `relations: { tenant: true }`.
So `Case.tenant`, `CaseHistory.tenant`, and `User.tenant` had the same
bug, latent.

**Fix.** All four `@JoinColumn` declarations now use the matching
camelCase name (`userId`, `tenantId`), so TypeORM treats the FK as
the same column as the `@Column` scalar — no phantom snake_case column.

**Files touched.**

- `backend/src/auth/entities/refresh-token.entity.ts` — `user_id` → `userId`.
- `backend/src/cases/entities/case.entity.ts` — `tenant_id` → `tenantId`.
- `backend/src/cases/entities/case-history.entity.ts` — `tenant_id` → `tenantId`.
- `backend/src/users/entities/user.entity.ts` — `tenant_id` → `tenantId`.
- `scripts/dev.sh` — `cmd_refresh` now distinguishes 500 (server bug)
  from 401 (real auth rejection) so the next time something like this
  fires, the script's error message points the right direction.

**Required local action.** Dev DB carries phantom `tenant_id` / `user_id`
columns from before this fix. `synchronize: true` won't drop them
automatically. Run `scripts/dev.sh db:reset` and restart `yarn dev:backend`
so TypeORM recreates the schema cleanly.

**Reference migration (`docs/reference/migrations/`)** was hand-written
with the *snake_case* names that matched my buggy `@JoinColumn`s. When
P10 reintroduces migrations-as-authority, the reference migration also
needs renaming to camelCase before it's revived. Flagged.

**Lesson.** Don't mix snake_case `@JoinColumn` names with camelCase
`@Column` property names. The two-column-phantom bug stays invisible
until the first endpoint that loads the relation. Tests against
mocked repos never see it. Caught by Adminer in seconds; surfaced in
prod as a 500.

---

## Mid-flight 9 — Vite proxy + CloudFront behavior allowlist drift

**Date:** 2026-05-19 (caught during P1.7 chunk C local test).

**Symptom.** Browser fired `GET /tour-state` → 200 with ~0.8 kB body
that wasn't JSON. Engine bootstrapped without errors. First step's
tooltip opened normally. Clicking through the tour fired
`POST /tour-state` → 404. No backend log entry for either request.

**Root cause.** The Vite dev proxy at `frontend/vite.config.ts` is an
explicit allowlist — `/auth`, `/cases`, `/users`. Anything not listed
falls through to Vite's SPA-fallback handler, which returns
`index.html` for GET (status 200, content HTML) and 404 for POST
to unknown paths. The frontend's `toursApi.list()` happily JSON-parsed
the HTML response — `JSON.parse('<!DOCTYPE html>...')` doesn't throw,
it just produces a weird tree the engine treated as "no tour records
yet" → tour fired correctly by accident. The POST then 404'd because
Vite returns 404 instead of falling through to the SPA for non-GET.

The same drift exists in CloudFront. `infra/modules/frontend_cdn/main.tf`
declares ordered behaviors for `/auth/*`, `/cases/*`, `/users/*` →
ALB. Anything else falls into the default behavior (S3 SPA) and then
the `custom_error_response` rewrites 404s to `/index.html` — the
same footgun that bit us in Pre-flight 22 / the P1.3 deploy. Two
proxies, same allowlist-drift pattern, both need updating every time
a new backend resource ships.

**Fix.**
1. `frontend/vite.config.ts` — added `/tour-state` to the proxy entries.
2. `infra/modules/frontend_cdn/main.tf` — added a fourth ordered
   `cache_behavior` block with `path_pattern = "/tour-state*"`
   targeting `alb-api`, methods + headers + cookies matching the
   existing three.
3. Cross-referenced this Mid-flight from both files' comments so the
   next person adding a backend resource sees the warning.

**Standing rule going forward.** Every new top-level backend resource
prefix is a three-touchpoint change:

- NestJS controller (`@Controller('<resource>')`)
- Vite proxy entry (local dev)
- CloudFront ordered cache behavior (deployed env)

Missing #2 produces a 404-on-POST or 200-but-HTML-on-GET in local
dev. Missing #3 produces the same in the deployed env. The symptom
is identical across both — diagnostic literacy says check the
response `Server` header (S3 = wrong) and the Content-Type
(text/html = wrong).

**Alternatives considered.**

1. **Refactor to a `/api/*` global prefix.** Nest would set
   `app.setGlobalPrefix('api')`, axios baseURL becomes `/api`, Vite
   proxies the single `/api/*` path, CloudFront has one behavior.
   Cleanest long-term, but it's a multi-file change touching every
   existing route prefix and would require a redeploy. Worth doing
   at the start of a phase, not mid-deploy of P1.7. Captured as a
   follow-up under "tech debt to revisit before P12."
2. **Wildcard proxy at Vite level (`/api` or `/^\/(?!@)/`).** Catches
   everything except Vite's own asset paths. Brittle — Vite's HMR
   namespace and dev-server endpoints (`/@vite/...`) need explicit
   carve-outs and the pattern is hard to audit. Not worth the
   cleverness right now.

**Revisit if.** A third new resource ships before P12 — three
similar incidents = three signals it's time for the `/api/*` prefix
refactor.

---

## Mid-flight 10 — RTK thunk dedup + Express ETag defaults (P1.8)

**Date:** 2026-05-19 (caught during P1.8 local test).

**Symptoms.** Two failures surfaced in the same debug session:

1. Dashboard rendered with widgets in empty state. **No `GET /cases`
   request fired** even though both data-widgets dispatched
   `fetchCasesThunk` on mount.
2. `GET /tour-state` returned **304 Not Modified** on every page load.
   Sometimes the tour fired correctly, sometimes the engine appeared
   to not evaluate triggers at all.

**Root causes (two separate bugs, same debug session).**

### 1. RTK thunk dedup check ran AFTER `pending` reducer

The first iteration of `fetchCasesThunk` put the "already loading or
already fetched, no-op" check inside the thunk body:

```ts
createAsyncThunk('cases/fetch', async (_, { getState }) => {
  const state = getState().cases;
  if (state.loading || state.fetched) return null;
  // ... fetch
});
```

Redux Toolkit dispatches the `pending` action **before** running the
thunk body. The pending reducer sets `state.loading = true`. By the
time the thunk body's `getState().cases` reads `state.loading`,
it's already `true` — so the check evaluates true and the thunk
no-ops on every dispatch. Net result: zero API calls, ever.

The correct pattern is `createAsyncThunk`'s `condition` option,
which runs BEFORE the pending action is dispatched and can return
`false` to skip the entire lifecycle:

```ts
createAsyncThunk('cases/fetch', asyncFn, {
  condition: (_, { getState }) => {
    const state = getState().cases;
    return !(state.loading || state.fetched);
  },
});
```

**Lesson.** When implementing RTK lazy-load thunks, the dedup goes
in `condition`, never in the thunk body. The reason is timing — the
RTK lifecycle dispatches pending first, body second, fulfilled
third. Any state mutation inside pending is visible to the body.

### 2. Express ETag defaults caused mysterious 304 responses

NestJS uses Express under the hood. Express has `etag` enabled by
default for GET responses. The browser caches the response body
keyed by ETag, and on subsequent requests sends `If-None-Match`;
the server returns 304 with an empty body, and the browser is
expected to serve the cached body from disk.

This is fine for static assets. For API endpoints that return
per-user mutable state (`/cases`, `/tour-state`, `/users`), it
creates two failure modes:

- A 304 with an empty body, if axios doesn't have the cached body
  for some reason (cross-session, dev-mode hot reload, browser
  cache cleared), returns `r.data` as `undefined`. Reducers that
  iterate over `r.data` crash silently, leaving the slice in an
  inconsistent state (`loading=false`, `fetched=false`, items
  empty) — exactly the no-op observed above.
- Cached bodies can go stale across user sessions if etags don't
  invalidate (e.g., the user switches accounts; the old user's
  response gets served to the new user from cache until the
  browser revalidates).

**Fix.** Disabled Express's automatic etag in `backend/src/main.ts`:

```ts
app.getHttpAdapter().getInstance().disable('etag');
```

API endpoints now always return fresh 200 responses with full
bodies. Frontend reducers don't need to handle the empty-body case
(though the tours `byTourId` helper still defensively handles
non-array input as belt-and-braces).

**Defensive complement.** Frontend reducers that iterate over a
fetched payload should still guard against undefined/non-array
input. The cost is one line; the bug class it prevents (silent
reducer crash → silent slice corruption → mystery empty-state UI)
is brutal to diagnose without it.

### Standing rules going forward

1. **RTK lazy-load thunks use `condition`, not in-body checks.** If a
   future thunk needs deduplication or any pre-dispatch gating, the
   pattern is the condition option.
2. **API responses are not cacheable.** etag stays off. If a future
   endpoint legitimately needs caching (e.g., a static configuration
   payload), it sets `Cache-Control` explicitly rather than relying
   on Express defaults.
3. **Reducers that iterate payloads defend against non-array input.**
   One `if (!Array.isArray(x)) return defaultValue;` guard at the top
   of every reducer that iterates a response.

**Revisit if.** A real performance requirement demands HTTP caching
of API responses. At that point, use explicit `Cache-Control:
private, max-age=...` on the specific endpoints that benefit,
rather than re-enabling etag globally.

---

## Mid-flight 11 — CloudFront path patterns: `/resource*` not `/resource/*`

**Date:** 2026-05-20 (caught immediately before the Article 01
LinkedIn launch — deployed dashboard returned no case data).

**Symptom.** `GET https://caseflow.musto.io/cases` returned 200 with
`Content-Type: text/html` and the SPA's `index.html` as the body.
The frontend Dashboard's `MyOpenCases` + `RecentActivity` widgets sat
in their empty states with no error message — the response parsed as
"not an array," and the slice gracefully fell back to no items.

**Root cause.** The original CloudFront ordered cache behaviors used
the pattern `/auth/*`, `/cases/*`, `/users/*`. CloudFront's glob
matches `*` as zero-or-more characters AFTER the slash — so
`/cases/*` matches `/cases/foo` but **does not match `/cases`
itself**. The bare-collection endpoint fell through to the default
behavior (S3), S3 returned 404, the distribution's
`custom_error_response` rewrote the 404 to `/index.html` → 200 OK
with HTML.

The lurking bug had been present since P10.6 (the original frontend
CDN module). It didn't surface until P1.8, when the Engineer
Dashboard became the first surface to call `GET /cases` (a bare
collection). Earlier calls were all `POST /auth/login`,
`POST /cases` (which CloudFront treats differently — POSTs are
forwarded based on the behavior's `allowed_methods`, and the
path-pattern match for the same path can behave subtly differently
across versions of the CloudFront engine).

The `/tour-state*` behavior I added during the Mid-flight 9 fix used
the correct no-slash pattern by accident — I'd written `/tour-state`
without a subpath in mind. That's why tours worked and cases didn't.

**Fix.** All four ordered cache behaviors now use
`<resource>*` (no slash before the asterisk):

- `/auth/*` → `/auth*`
- `/cases/*` → `/cases*`
- `/users/*` → `/users*`
- `/tour-state*` (unchanged — was already correct)

After `terragrunt apply` in `infra/live/dev/frontend/` and the
~5–10 min CloudFront propagation, the bare-collection routes work.

**The deeper lesson — diagnostic shape repeats.** This is the **same
diagnostic signature** as Mid-flight 9:

- Response status 200
- `Content-Type: text/html`
- `Server: AmazonS3`
- Body starts with `<!doctype html>`

That's the universal "CloudFront served the SPA instead of routing
to the ALB" signature. Whenever it appears, the cause is one of:

1. **No matching ordered behavior** for the requested path
   (Mid-flight 9 — we forgot to add the behavior at all).
2. **Behavior exists but pattern is wrong** (this Mid-flight 11 —
   `/resource/*` doesn't match bare `/resource`).
3. **Backend itself returned 404** (Pre-flight 22-era P1.3 deploy —
   the ECS task was running an older image without the route).

The diagnostic literacy compounds. Next time we see this shape, the
first question is "which of the three is it?" — not "what's broken?"

**Standing rule reinforcement.** The Mid-flight 9 three-touchpoint
rule (NestJS controller + Vite proxy + CloudFront behavior) gets a
clarifying sub-rule:

> **CloudFront path patterns for API routes use `<resource>*`, not
> `<resource>/*`.** The single-asterisk form matches both the bare
> collection and all subpaths. The slash-asterisk form matches only
> subpaths, leaving the bare collection unrouted — which is invisible
> until the day someone calls the bare endpoint.

**Revisit if.** A future refactor moves all API routes behind an
`/api/*` global prefix (deferred from Mid-flight 9). At that point
one ordered behavior replaces all four, and the path-pattern shape
becomes a single decision instead of four.

---

## Pre-flight 23 — In-app guidance: tour engine architecture (P1.7)

**Date:** 2026-05-19 (P1.7 opening).
**Decision.** Build an internal-only, login-gated tour engine on top of
**react-joyride** with the following architecture:

- **Targeting** — every interactive UI element gets a `data-tour-id="domain.element"`
  attribute (e.g. `cases.create-button`, `dashboard.my-open-cases`). Tours
  target by `data-tour-id`, never by CSS class or component name. The
  attribute survives Tailwind/MUI restyles and component refactors.
- **Tour definitions** — declarative TypeScript modules under
  `frontend/src/tours/`, one file per feature. Shape: `{ id, version,
  audience: Role[], trigger: 'first-login' | 'manual' | 'event:<name>',
  steps: TourStep[] }`. Definitions ship in the same PR as the feature
  they describe.
- **Tour engine** — a single `<TourEngine />` mounted inside `<Layout />`
  (only reachable behind `RequireActiveAuth`). The engine reads per-user
  tour state from Redux, decides which tours to offer, and renders
  react-joyride's overlay with MUI-styled tooltip bodies.
- **Audience gate** — engine renders only for `role: ENGINEER | ADMIN`.
  Never renders for `role: CUSTOMER`. Hard rule enforced inside the
  engine, not in route guards, so the gate stays close to the rendering
  decision.
- **State persistence** — server-side via a new `UserTourState` entity
  (`userId`, `tenantId`, `tourId`, `version`, `status:
  started|completed|dismissed`, `completedAt`). Two endpoints:
  `GET /tour-state` returns the caller's record; `POST /tour-state`
  upserts a single tour's status. Tour *state* lives in DB; tour *content*
  stays source-controlled.

**Alternatives rejected.**

1. **driver.js instead of react-joyride.** Lighter, framework-agnostic,
   newer API. Rejected for now because react-joyride composes cleanly
   with our MUI 7 + React Router 7 stack and the type definitions are
   ahead of driver.js. Revisit if bundle size becomes a measured concern.
2. **Build from scratch.** ~3 weeks of work for marginal control. The
   thing we'd be building is `react-joyride` minus type maintenance.
3. **Tour content in the database with an admin CRUD UI.** Looks like
   flexibility, ends up being a CMS we have to maintain. Source-controlled
   content ships with the feature PR, gets code review, versions in
   git, and never goes out of sync with the UI it describes. Revisit
   only if a per-tenant customization requirement lands.
4. **localStorage-only state.** Zero backend work but doesn't sync across
   devices. The senior call for a SaaS is server-side state — a single
   table, two endpoints, syncs forever.
5. **CSS-class targeting.** Breaks on every theme tweak. The
   `data-tour-id` attribute survives.

**Internal-only constraint.** Tours never render for `CUSTOMER` users.
This is decided in the engine, not the route guard, so the rule is
visible at the rendering boundary. Future customer-facing onboarding
(if and when P14 customer app lands) gets its own engine with its own
audience rules — sharing the engine across audiences would force a
permission model we don't need yet.

**Versioning.** Each tour carries a `version`. Bumping the version
re-shows the tour to users who'd already completed the prior version,
making "what's new" announcements free. Audit-friendly: which version
did this user see?

**i18n.** Tour copy is plain string literals in v1, not yet wrapped in
an i18n hook. The `TourStep` body type is shaped to accept either a
string or a translation key, so adding i18n later is a per-field swap,
not a redesign.

**Revisit if.** Bundle weight from react-joyride becomes measurable in
LCP; OR a customer-facing onboarding requirement lands; OR per-tenant
tour customization becomes a real product requirement.

---

## Pre-flight 24 — Engineer Dashboard composition (P1.8)

**Date:** 2026-05-19 (P1.8 opening).
**Decision.** Replace the placeholder `<Home />` page with a **widget-
composed Engineer Dashboard** at `/`. The dashboard is the post-login
landing surface for `ENGINEER` and `ADMIN` users. Composition rules:

- **Widget registry pattern.** Each widget is a React component plus a
  registration entry that declares its `id`, `title`, `dataSource`
  (which existing endpoint feeds it), and `audience` (roles that see
  it). The dashboard page reads the registry and renders the widgets
  the current user is authorized for, in a stable order.
- **MVP widgets (P1.8 ships these three):**
  - `myOpenCases` — "Cases assigned to me, by priority, by age." Reads
    `/cases?assignedToMe=true&status=OPEN_*`. Empty state: "No cases
    assigned. Browse all cases →".
  - `recentActivity` — "Latest tenant activity." Reads from the
    existing `CaseHistory` records, latest 20.
  - `quickActions` — "Create case" button + global search box (search
    is stubbed for v1, wired up in P5 or later).
- **Progressive enrichment.** Each subsequent phase adds widgets to
  the registry without touching the page shell:
  - P2 adds `customersIWork` (counts + recent customer interactions).
  - P3 adds `recentNotes` and `mentionsOfMe` (when @-mentions land).
  - P5 adds `slaAtRisk` (cases breaching within X hours) and
    `breachedToday` (counter).
  - P6 adds `myQueues` (depth per queue I belong to) and
    `workloadSummary` (the manager-targeted projection).
  - P8/P9 add `aiArtifactsForReview` (pending-approval AI outputs).
- **Customer route.** `role: CUSTOMER` never sees this dashboard.
  Routing handles it: a separate `<CustomerDashboard />` at the same
  path renders based on role (P14 will formalize this). For now,
  `CUSTOMER` users land on the existing `<Home />` placeholder.

**Alternatives rejected.**

1. **Monolithic dashboard page that grows feature-by-feature.** Every
   new widget means editing the dashboard page directly, which
   becomes a merge-conflict surface and an authorization-logic mess.
   Registry pattern keeps the page shell stable.
2. **Server-rendered dashboard view (single endpoint returning a
   composite payload).** Tempting for performance, premature for now.
   Each widget hits its own existing endpoint; React Query (or our
   own caching) handles parallelism. Revisit when dashboard load time
   becomes a measured concern.
3. **Defer the dashboard until P6 (when workload + queues land).**
   Rejected because the post-login surface is what every demo viewer
   sees first. A placeholder Home page makes the project look unfinished
   even when the backend is fully operational. The MVP dashboard at
   P1.8 ships with three widgets and grows from there.

**Empty-state UX.** The MVP launches against a system with potentially
zero cases per tenant (especially for newly-approved users). Every
widget needs a designed empty state — "No cases yet. Create your
first →" — rather than a blank panel. The demo seed harness (Pre-flight
25) prevents this in demo environments; the empty state covers
real-user first-runs.

**Anti-goal.** Widgets do NOT introduce new backend endpoints in
P1.8 — they consume what already exists from P1.1-P1.5. New endpoints
land in the phase whose entities they expose (P2 for customers, etc.).

**Revisit if.** Performance becomes a real concern (compose at the
API layer); OR widget logic grows complex enough that the registry
pattern starts feeling heavy (lift into a dedicated widget framework
later).

---

## Pre-flight 25 — Demo seed harness (P1.9)

**Date:** 2026-05-19 (P1.9 opening).
**Decision.** Build a **single-command, idempotent, deterministic
demo seed** that produces a presentable dataset for local development
and for the deployed dev environment. Shape:

- **Entry points.**
  - `yarn db:seed:demo` — runs against the local Postgres.
  - `scripts/dev.sh demo:seed` — wraps the yarn command + adds an
    explicit `--target=local|rds` flag and a confirmation prompt for
    `rds`. Belt-and-braces against accidentally seeding prod-shaped
    data into the wrong DB.
- **Idempotency.** Re-running wipes the seed-managed rows (identified
  by a `seed_origin` marker column or by tenant slug prefix) and
  re-creates them from scratch. User-created data not flagged as
  seed-origin stays intact.
- **Determinism.** Same inputs (same `SEED_VERSION`) produce
  byte-identical output every run. Cases get fixed UUIDs derived from
  a hash of (tenant slug + case number + seed version). No `Math.random()`
  in the seed path — RNG calls go through a seeded PRNG with the seed
  version as input.
- **Dataset shape (P1.9 ships these):**
  - 3 tenants: `abc-company` (the default), `acme-services`, `globex-support`.
  - Per tenant: 1 admin, 3 engineers, 2 agents, 4 customers
    (all `ACTIVE` for ease of demoing; one `PENDING_APPROVAL` per
    tenant to demo the approval flow).
  - Per tenant: ~35 cases across statuses, priorities, and ages. Cases
    pre-distributed across engineers so the "my open cases" widget
    has content for every demo user.
- **Demo credentials.** Documented in `scripts/README.md` under a
  clearly-marked "DEMO ONLY" header. Every demo user has a known
  password — e.g. `demo123!`. The seed refuses to run in `NODE_ENV=production`
  (process-level guard, not config-level).
- **Progressive enrichment.** Each subsequent phase extends the seed:
  - P2 adds customer organizations + contacts, re-links cases to them.
  - P3 adds notes (mix of internal + customer-visible) + richer history.
  - P5 adds SLA fields + a handful of breached + at-risk cases.
  - P8 adds pre-approved AI artifacts and a few pending-review ones.

**Alternatives rejected.**

1. **Inline seed in `TenantsService.onModuleInit`.** That hook already
   exists for the default tenant, but extending it for a full demo
   dataset confuses two concerns: bootstrapping a usable system
   (always-on) vs creating a demo dataset (developer-triggered). The
   demo seed is a separate script.
2. **Fixtures via TypeORM seeders.** Heavyweight, opaque, hard to
   evolve. A plain TypeScript script with the entity classes is
   simpler and stays readable.
3. **Faker-generated data.** Non-deterministic by default. We need
   the same cases in the same statuses every demo so the screenshots
   in the content series stay accurate. Demo dataset is hand-curated.
4. **Skip the RDS-target path; seed only local.** Rejected because the
   deployed `caseflow.musto.io` IS the portfolio surface. Anyone
   clicking the link from the LinkedIn post needs to see a populated
   system, not three accounts and zero cases.

**Anti-goals.**

- The seed script must NOT create data that exercises code paths not
  yet shipped in production. P1.9 seeds users + cases. P2 seeds
  customers when those entities exist. Trying to seed "what we plan
  to ship" produces orphaned rows and broken UIs.
- The seed must NOT have any non-deterministic dependencies (random
  UUIDs, `Date.now()`, network calls). Demo environments must be
  reproducible.

**Reset semantics.** `dev.sh demo:reset` is shorthand for:
1. Truncate seed-origin rows.
2. Re-run the seed against the current DB.

Useful as the last step of a demo run-through ("OK, let me reset and
show you again from scratch") and as the first step before recording
a demo video.

**Revisit if.** The demo dataset grows large enough that the seed
takes more than 10s to run; at that point split into a "fast" minimal
seed (1 tenant, 5 cases) and a "full" demo seed.

---

## Pre-flight 26 — User identity fields in JWT claims (no `/users/me` endpoint)

**Date:** 2026-05-20 (P1.8 follow-on; surfaced when the Engineer Dashboard
needed to display the user's full name in the welcome heading).

**Decision.** Display fields (`name`, `email`) carried as additional
JWT claims alongside the already-signed authorization claims
(`sub`, `tenantId`, `role`, `status`). No `/users/me` endpoint
introduced. Frontend `AuthUser` shape extended with `name` + `email`;
`decodeAccessToken` propagates them; the dashboard reads from
`auth.user.name` like any other claim.

**Why this path (Path A from the JWT-vs-fetch trade-off).**

1. **Backend already signs claims at login.** Adding two fields to
   `tokenSubjectFor` / `issueTokenPair` is a one-line change. No new
   endpoint, no new controller test, no new frontend slice, no
   bootstrap-time additional HTTP call.
2. **Single source of truth for the session.** Whatever the backend
   signed into the JWT is the user's effective identity for that
   session. Frontend and backend agree on the same shape; no
   possibility of the JWT saying one thing and `/users/me` saying
   another (which would otherwise create a 15-minute window of
   inconsistency).
3. **Portfolio MVP context.** This is not a regulated SaaS where PII
   in tokens triggers compliance review. Tokens are short-lived
   (15 min), the audience is internal-only (dashboard never renders
   for `CUSTOMER`), and the project ships in a single AWS dev account.
   The Path B trade-off (extra HTTP call + new endpoint + new slice)
   doesn't pay for itself yet.

**Path B (deferred) — `/users/me` endpoint + dedicated profile fetch.**
Considered and rejected for now. The path would:
- Add `GET /users/me` returning the full safe-user shape
- Add a `usersSlice` to the frontend with a `fetchMeThunk`
- Fire the fetch in parallel with the existing `refreshThunk` during
  session bootstrap
- Require the dashboard to wait for both before rendering name

Path B is the right answer when:
- Profile data grows past what fits in a JWT (avatars, preferences,
  rich metadata)
- Name changes need to reflect within seconds instead of 15 minutes
- Regulatory context demands that PII not ride on every request
- A non-browser client (CLI, mobile) needs profile data without
  decoding the token

**Pattern crystallized — JWT carries authorization, profile fetch
carries display.** The discipline that emerged from this decision:

| Field | JWT-decode (Path A) | Fetch endpoint (Path B) |
|---|---|---|
| `sub` / `userId` | ✅ guards need it | overkill |
| `tenantId` | ✅ guards need it | overkill |
| `role`, `status` | ✅ guards need it | overkill |
| `name`, `email` | ✅ (P1.8 chose Path A) | acceptable |
| `picture`, `preferences`, etc. | ✗ — too large, too mutable | ✅ |

The rule: **JWT carries everything required to *authorize* a request.
Everything required only to *display* the user goes through a fetch
when fetch-time semantics actually matter.** For the current scope,
"name + email + role + status" all sit comfortably in the JWT.

**Revisit if.**

1. Profile data extends beyond name/email (avatar URL, timezone,
   notification prefs, etc.). Adding fields one-by-one to the JWT
   becomes a slippery slope; that's the moment to introduce
   `/users/me` and demote `name + email` to the fetch path.
2. A real product requirement demands sub-15-minute freshness on
   name changes (e.g., admin rename in the UI showing immediately).
3. Regulatory context lands (PHI, financial data, GDPR-sensitive
   surfaces) where PII-in-tokens triggers compliance review.

**Files touched at decision time.**

- `backend/src/auth/auth.service.ts` — `issueTokenPair` signs `name`
  and `email` alongside the existing claims
- `frontend/src/features/auth/types.ts` — `DecodedAccessToken` and
  `AuthUser` extended with `name + email`
- `frontend/src/features/auth/jwt.ts` — `decodeAccessToken` reads
  the new fields
- `frontend/src/pages/Dashboard/Dashboard.tsx` — welcome heading
  reads `user.name` instead of `user.role.toLowerCase()`
- `frontend/src/features/tours/TourEngine.test.tsx` — mock
  `AuthUser` fixture updated to include the new fields

---

## Pre-flight 27 — Showcase repo pattern over fully-public main repo (P1 close-out)

**Date:** 2026-05-21 (P1 close-out, alongside the v0.1.0 tag).

**Decision.** Keep the implementation repo (`caseflow-ai`) **private**.
Build a curated public companion at
[`caseflow-ai-showcase`](https://github.com/amusto/caseflow-ai-showcase)
that contains only the methodology surface: the ROADMAP, the decision
log, the dev-ready specs, the AI-Ready SDLC graphic, the long-form
articles as they ship, and 3–4 screenshots of the deployed product.
The showcase repo's `README.md` is the portfolio hero; the private
repo's `README.md` reverts to a developer-onboarding shape (quick
start, helper scripts, "where to start reading" for internal devs).

**Alternative rejected — fully public main repo.** The default
engineering-portfolio move is to flip the whole thing public on day
one. The reasoning is "anyone should be able to clone and verify the
code." Three problems with that for this project:

1. **The audience doesn't operate that way.** Recruiters and
   engineering leaders don't clone repos — they read READMEs, scan
   roadmaps, look at architectural decisions. Optimizing for
   clone-and-verify burns effort the audience never uses.
2. **Senior engineers are evaluated on judgment, not code quality.**
   Which problems got chosen, what was cut, where the team reversed
   course, how it got documented. The methodology artifacts carry
   that signal at far higher density than `git log --all -p`.
3. **The implementation repo carries operational metadata that
   shouldn't be public.** Account IDs in Terraform outputs, RDS
   endpoints, S3 bucket names, internal scripts (`./scripts/dev.sh`,
   `./scripts/deploy.sh`) that assume specific AWS account access.
   Sanitizing all of that to publish-grade adds toil at every commit.

**Alternative rejected — third-tier public excerpts folder
(`docs/patterns/`) from day one.** Considered carving curated 10–20
line code excerpts (widget registry contract, role-gate-inside-the-
engine, RTK `condition` pattern, deterministic UUIDs, Terragrunt
layering) into a `docs/patterns/` folder in the showcase. Decided
against shipping it in the P1 close-out drop because:

- It risks blurring the line between "methodology surface" and
  "partial source dump"
- The patterns are most legible when paired with the surrounding
  context (modules, types, tests), which the excerpts can't carry
- The decision log already cites the relevant files by path, so the
  excerpts add less signal than they imply

Deferred to a Phase 2 close-out update if recruiters or engineering
leaders consistently ask "can I see a representative code shape?".

**Why this works.**

- **Mirrors real production codebases.** Serious orgs keep their
  source private; methodology + architectural artifacts are what
  they share externally (design docs, postmortems, conference talks).
- **The split shows judgment, not just output.** The fact that there
  *is* a deliberate split, with reasoning, is itself a signal. A
  recruiter or hiring manager who reads this entry has already learned
  something about how I think.
- **Lower toil per phase close-out.** A 5-minute "sync the
  methodology artifacts" ritual at each phase close-out keeps the
  showcase fresh; the implementation repo doesn't have to be polished
  for public consumption on every commit.
- **The live demo carries the verifiability burden.** Anyone can
  register at `caseflow.musto.io`, get approved, and drive the
  product — that's a stronger validation of "the code works" than
  any clone-and-build dance.

**Revisit if.**

1. Recruiters or engineering leaders consistently ask for code
   access — would suggest the methodology artifacts aren't carrying
   the signal and the excerpts folder needs to land.
2. The methodology artifacts get stale because the implementation
   has drifted — would suggest tightening the phase-close-out sync
   ritual or automating it.
3. A future phase produces a pattern compelling enough on its own
   that it deserves a runnable public demo (e.g., a self-contained
   library or pattern repo) — would be additive to the showcase,
   not a replacement.

**Files touched at decision time.**

- `caseflow-ai/README.md` — reverted to developer-onboarding shape
  (private-repo-internal audience); links to the showcase repo in
  three places for anyone with read access who wants the public-
  facing context
- `caseflow-ai/docs/screenshots/` — source-of-truth screenshots
  staged for the showcase (copied via SETUP.md ritual)
- `caseflow-ai-showcase/` (new public repo) — hero README, ROADMAP,
  decision log, specs, architecture, Article 01, 3 screenshots,
  LICENSE (CC BY 4.0)
- `publishing/showcase-repo-draft/` — staging folder used to compose
  the initial drop; retired after the live repo went up
- `publishing/demo-invitation-language.md` — three forms (long /
  medium / short) of the "register + DM me" demo invitation,
  reused across the showcase README, Article 01 footer, and a
  standalone LinkedIn post

---

## How this file is updated

1. Pre-flight decisions are written before any sub-phase code lands.
2. New decisions discovered mid-phase get appended as `Mid-flight N —`
   sections with the same shape (Decision / Alternative rejected /
   Revisit if).
3. When P1 closes (sub-phases all ✅), this log's status flips to
   **Closed** and a one-line summary lands in the ROADMAP "Where to
   look" cell for P1.
4. Subsequent phases get their own `phase-NN-<slug>.md` log.
