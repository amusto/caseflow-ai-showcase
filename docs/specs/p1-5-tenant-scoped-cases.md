# Dev-Ready Task — P1.5 · Tenant-scoped cases queries

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.5**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Close the multi-tenancy loop. Every cases query now scopes by
`user.tenantId`. Tenant A users cannot read, update, or even confirm
the existence of Tenant B's cases. Also fixes a latent NOT-NULL bug
in P1.1's schema: `CasesService.create` and `recordHistory` never set
`tenantId`, so any real `POST /cases` would error with a Postgres
not-null violation. Both fixes land together.

## Business / user value

This is the data-isolation promise made by P1.1's schema and P1.4's
auth gate. Without it, the multi-tenant SaaS positioning is false on
its face: any authenticated user could read every case in every
tenant. P1.5 makes the boundary real at the data-access layer.

## Constraints

- **Architecture:** `CasesService` reads `user.tenantId` from
  `RequestUser` (already populated by P1.4's guards). Every persistence
  call includes `tenantId`. No new dependencies, no new endpoints.
- **Security:** Cross-tenant access returns 404 (per Pre-flight 12),
  not 403 with "wrong tenant" — avoids existence-leak.
- **Compatibility:** P1.2's auth contract unchanged. P1.4's guards
  unchanged. Only `CasesService` internals and tests change.
- **Delivery:** Conventional Commit:
  `feat(backend): tenant-scoped cases queries (P1.5) + fix latent NOT NULL`.

## Affected areas

**Service:**
- `backend/src/cases/cases.service.ts` —
  - `create`: set `tenantId: user.tenantId` on the new Case; pass
    `tenantId` to `recordHistory`.
  - `findAll`: add `tenantId: user.tenantId` to the where clause for
    both CUSTOMER and ENGINEER/ADMIN paths.
  - `findOne`: add `tenantId: user.tenantId` to the where clause.
    Cross-tenant access produces `null` from `findOne`, which already
    throws `NotFoundException` — no code path change needed for the
    404 semantic.
  - `recordHistory`: new `tenantId` parameter. Internal callers pass
    it from `user.tenantId`.
- `update` works by virtue of calling `findOne` first — no separate
  change needed.

**Tests:**
- `backend/src/cases/cases.service.spec.ts` — add cross-tenant
  isolation cases:
  - User from Tenant A creates a case → `tenantId` matches user's.
  - User from Tenant A reads their own case → 200.
  - User from Tenant A tries to read Tenant B's case → 404
    (NotFoundException), **not** 403.
  - User from Tenant A tries to update Tenant B's case → 404.
  - Existing happy-path tests adjusted for the new `RequestUser` shape
    (already done in P1.4; verify still passing).

**Out of scope (deliberate):**
- Cross-tenant `assignedTo` validation (an admin shouldn't be able to
  assign a Tenant A case to a Tenant B user). Small but pulls
  `UsersService` into `CasesService` — defer to a P1.5b or a future
  hardening sub-phase. Tracked.
- `CaseHistory` having its own `tenantId` denormalized vs derived from
  the parent Case. P1.1 schema chose denormalized. P1.5 follows that
  choice (writes `tenantId` on every history row). Not revisiting now.
- Frontend changes — P1.6 territory.

## Acceptance criteria

- [ ] `curl POST /cases` (as an ACTIVE user) succeeds with a 201 + the new case carrying the user's tenantId.
- [ ] `curl GET /cases` (as Tenant A user) returns only cases where `tenantId = Tenant A`. Tenant B's cases never appear.
- [ ] `curl GET /cases/<tenantBCaseId>` (as Tenant A user) returns **404 Not Found**, not 403.
- [ ] `curl PATCH /cases/<tenantBCaseId>` (as Tenant A user) returns **404 Not Found**.
- [ ] `case_history` rows carry the correct `tenantId` from creation onward.
- [ ] `yarn workspace @caseflow-ai/backend lint && typecheck && test` clean.

## Test plan

- **Unit:** extend `cases.service.spec.ts` with the cross-tenant
  isolation cases listed above. Mocked repository, so testing the
  where-clause shape via `expect(caseRepository.find).toHaveBeenCalledWith({ where: { tenantId: '...' }, ... })`.
- **Manual:** create two tenants in Adminer, register a user under each
  (promote both to ACTIVE), log in as each, attempt cross-tenant access.

## Security / privacy notes

- 404-not-403 for cross-tenant access is the production-grade pattern.
  Returning 403 leaks "this case exists, you just can't see it" — a
  real attacker can enumerate case IDs across tenants.
- `tenantId` is set server-side from the authenticated user's token,
  never accepted from request bodies. The DTO has no `tenantId` field.

## Rollback plan

Single commit. Revert with `git revert <sha>`. No schema change, no
data migration — pre-flight 3's wipe-and-reseed is still the canonical
escape hatch for dev DB shape problems.

## Resolved pre-flight decisions

Captured in the decision log:

- **Pre-flight 12** — cross-tenant access response shape: **404 Not Found**
  (not 403). Rejected: leaking tenant boundaries to debug-friendly
  error codes.

## How we'll work this sub-phase

Per Option 1: agent drives implementation, user reviews diffs.
