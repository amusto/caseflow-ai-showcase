# Dev-Ready Task — P2.4 · Customer CRUD + `CustomerOrganizations` admin widget

**Phase:** [P2](../../../../ROADMAP.md#phase-2--admin-section--customer-model-) · Sub-phase **P2.4**
**Decision log:** [`docs/decisions/phase-02-admin-and-personalization.md`](../../../../docs/decisions/phase-02-admin-and-personalization.md) — Pre-flights 12–14 (drafted alongside this spec; see "Resolved pre-flight decisions" below)
**Status:** dev-ready

## Goal

Build the backend CRUD layer on top of the P2.3 customer entities and add the second admin widget — `CustomerOrganizations` — to the `/admin` surface. The widget exists primarily to demonstrate the widget-registry pattern accumulating without touching the page shell (the architectural payoff promised in P2.1's Pre-flight 2 finally pays off).

P2.4 splits sharply between layers. **Backend ships full CRUD** for both `CustomerOrganization` and `CustomerContact` — list, create, read, update — because P2.5 (case FK + backfill) and the demo workflow both need real customer rows to exist via the API surface, not just via the seed. **Frontend ships only the read-side widget**; the full create + edit UI for engineers lives in P2.7 alongside the customer list / detail pages. The admin widget's whole job is "prove the registry accumulates" — it's a read-only summary, not a CRUD console.

The P2.4 endpoint surface also serves P13's email-in flow long-term: P13.3's sender matching reads from `customer_contacts`, and P13.4's draft-case workflow needs to be able to promote unmatched senders into new `CustomerContact` rows via the same endpoints P2.4 exposes here.

## Why this architecture (the 90-second mental model)

Three threads.

**Thread 1 — CRUD shape.** Standard REST controllers (`CustomersController` for orgs, plus contact sub-endpoints) backed by a single `CustomersService` that owns both entities. The service is the single guard for the cross-tenant FK invariant Pre-flight 11 documented but couldn't enforce at the DB layer (`customer_contacts.tenantId` must equal its parent org's `tenantId`). Every contact create + update routes through `CustomersService` which reads the parent org's tenant before persisting. Pre-flight 12 locks in the URL shape: nested for parent-context operations (`POST /customers/:orgId/contacts`, `GET /customers/:orgId/contacts`), flat for individual contact operations (`GET /contacts/:id`, `PATCH /contacts/:id`) since the contact's UUID is globally unique and the parent context isn't needed for those.

**Thread 2 — No DELETE in P2.4 (Pre-flight 13).** "Delete" against a customer org or contact has two issues: (a) hard-deleting an org cascades to its contacts (which is fine by itself but loses correspondent history); (b) once P2.5 adds `Case.customerOrganizationId`, a hard-deleted org would orphan its cases (or refuse to delete, depending on FK strategy). The clean way is to leave delete out entirely in P2.4. `PATCH /customers/:id { status: 'INACTIVE' }` covers the "we don't do business with them anymore" lifecycle. A future sub-phase can add hard delete once the case-FK story (P2.5) is settled and the right confirmation flow has a real home in the UI (P2.7+).

**Thread 3 — Primary contact stays denormalized; no auto-sync (Pre-flight 14).** Pre-flight 10 in P2.3 accepted that `customer_organizations.primaryContactEmail` can drift from the matching `customer_contacts.email`. P2.4 implements that by *not* writing a sync trigger when a contact is created or updated. The create-org flow offers an "auto-fill from first contact" affordance (the form copies the first contact's fields into the org's primary slots at submit time), but that's a one-shot fill, not a live binding. Anyone needing the canonical correspondent list reads `customer_contacts`; anyone needing the org's at-a-glance primary contact reads the org row.

**The widget.** `CustomerOrganizations` is a single MUI `<Card>` with a list of customer orgs sorted ACTIVE-first, then by name. Each row carries the org name, the denormalized `primaryContactEmail`, a contact-count chip, and a status chip. Empty state for tenants with no customer orgs yet. No add/edit buttons in P2.4 — the widget's job is composition proof; engineer-facing CRUD lives in P2.7. The slice (`customersSlice`) follows the same shape as `usersSlice` from P2.1: a single `fetchCustomersThunk` with RTK `condition` dedup, `createSelector`-memoized selectors, lazy-loaded on widget mount.

**What's deliberately NOT here.**

- No engineer-facing customer pages (list, detail, create / edit forms) — lands in P2.7.
- No DELETE endpoints — see Pre-flight 13.
- No `Case.customerOrganizationId` FK — lands in P2.5.
- No contact-to-user linking (a CustomerContact may also be a customer-portal User someday) — out of scope until P14 customer-app lands.
- No bulk operations (bulk-create contacts, bulk-deactivate orgs, etc.) — single-row only.
- No search/filter UI beyond the widget's ACTIVE-first sort. Engineer-facing list page in P2.7 picks up filter + search.

## Affected areas

### Backend — new files

**DTOs (`backend/src/customers/dto/`):**

- `create-customer-organization.dto.ts` — `name` (required, max 160), `primaryContactName` / `primaryContactEmail` / `primaryContactPhone` (optional, length-validated per the schema), `status` (optional, defaults to `ACTIVE` server-side). `@IsString` + `@MaxLength` + `@IsOptional` + `@IsEnum` as appropriate.
- `update-customer-organization.dto.ts` — all fields optional (partial update). Validates only the present fields.
- `list-customer-organizations-query.dto.ts` — `?status=ACTIVE|INACTIVE` (optional, `@IsEnum`). Absent = all statuses. Mirrors `list-users-query.dto.ts` from P2.1.
- `create-customer-contact.dto.ts` — `name` (required, max 120), `email` (required, RFC 5321 max 254, `@IsEmail`), `phone` (optional, max 32). Org id comes from the URL path (`:orgId`), not the body.
- `update-customer-contact.dto.ts` — all fields optional. Email update is allowed; service-layer enforces the composite unique re-check.

**Service + controller:**

- `customers.service.ts` — owns both entities. Methods (sketch):
  ```ts
  listOrgs(tenantId, filters): Promise<CustomerOrganization[]>
  findOrgById(orgId, tenantId): Promise<CustomerOrganization>  // 404 on cross-tenant
  createOrg(dto, tenantId): Promise<CustomerOrganization>
  updateOrg(orgId, dto, tenantId): Promise<CustomerOrganization>
  listContactsForOrg(orgId, tenantId): Promise<CustomerContact[]>
  findContactById(contactId, tenantId): Promise<CustomerContact>  // 404 on cross-tenant
  createContact(orgId, dto, tenantId): Promise<CustomerContact>   // enforces cross-tenant FK invariant
  updateContact(contactId, dto, tenantId): Promise<CustomerContact>  // enforces email-within-org uniqueness on email change
  ```
- `customers.controller.ts` — REST controller, `@Controller('customers')`. Endpoints listed under "API surface" below. Admin role NOT required for read endpoints (engineers need to read customer data to file cases against them); admin role IS required for create + update endpoints in P2.4 (engineers get write access in P2.7 if/when the UX surfaces it; the conservative default is admin-gated until the engineer flow ships).
- `contacts.controller.ts` — separate controller for the flat individual-contact endpoints (`GET /contacts/:id`, `PATCH /contacts/:id`). Pulling these out of `CustomersController` keeps the URL prefix matching the resource. Both controllers share the same `CustomersService`.

**Specs:**

- `customers.service.spec.ts` — covers all eight service methods + the cross-tenant FK invariant on create/update contact + email uniqueness re-check + status-INACTIVE filter.
- `customers.controller.spec.ts` — endpoint shape + tenant scoping + admin-only write gate.
- `contacts.controller.spec.ts` — endpoint shape + tenant scoping (cross-tenant returns 404) + admin-only write gate on PATCH.

### Backend — modified files

- `backend/src/customers/customers.module.ts` — add `CustomersService` to providers; add `CustomersController` + `ContactsController` to controllers; export `CustomersService` so future modules (`CasesModule` in P2.5) can inject it for case-FK validation.

No other backend file changes. Schema is set from P2.3.

### Frontend — new files

**Customers feature folder (`frontend/src/features/customers/`):**

- `customersApi.ts` — axios wrappers for the endpoints P2.4 ships. P2.4 only uses `list(filters?)` from the widget side; the others are stubbed for P2.7's consumption to keep the API surface consolidated. Same shape as `usersApi.ts`.
- `customersSlice.ts` — Redux slice. `fetchCustomersThunk(filters)` with RTK `condition` dedup. Selectors: `selectActiveCustomerOrgs(state)`, `selectCustomerOrgById(state, id)`. Listing-by-status uses a memoized selector pattern.
- `types.ts` — `CustomerOrganization` + `CustomerContact` wire shapes (mirrors backend entity but with `Date` fields as ISO strings).
- `index.ts` — public barrel.

**Widget:**

- `frontend/src/features/admin/widgets/CustomerOrganizations.tsx` — reads from `customersSlice`, filters by `status === ACTIVE`. Renders a `<List>` of `<ListItem>`s with org name (primary text), `primaryContactEmail` (secondary), a `<Chip>` for contact count, a status `<Chip>` (color="success" for ACTIVE). Cap at 10 visible rows; if more exist, show a footer link "View all (12 total) →" pointing at the P2.7 placeholder route. Empty state: "No customer organizations yet." Loading state: 3-row skeleton.

**Frontend — modified files:**

- `frontend/src/features/admin/registry.ts` — add `CustomerOrganizations` to the `adminWidgets` array.
- `frontend/src/store/index.ts` — register the `customers` reducer alongside the existing reducers.

**Frontend — new test files:**

- `frontend/src/features/customers/customersSlice.test.ts` — slice + thunk + selector coverage.
- `frontend/src/features/admin/widgets/CustomerOrganizations.test.tsx` — widget renders empty / loading / rows / "view all" footer; status filter works.

## API surface

```
GET    /customers                                — list orgs in actor's tenant, ?status filter
POST   /customers                                — create org (ADMIN only in P2.4)
GET    /customers/:orgId                         — get one org (404 if cross-tenant)
PATCH  /customers/:orgId                         — update org (ADMIN only in P2.4)

GET    /customers/:orgId/contacts                — list contacts for that org
POST   /customers/:orgId/contacts                — create contact under that org (ADMIN only)

GET    /contacts/:contactId                      — get one contact (404 if cross-tenant)
PATCH  /contacts/:contactId                      — update contact (ADMIN only in P2.4)
```

**Why this shape (Pre-flight 12).** Parent-context operations (`list contacts for this org`, `create contact under this org`) express the relationship in the URL — anyone reading the path knows the contact belongs to this org. Individual contact operations (`get this specific contact`, `update this specific contact`) don't need the parent context — the contact's UUID is enough. Fully-nested URLs (`/customers/:orgId/contacts/:contactId`) duplicate information that's redundant by the time you have the contact's id; fully-flat URLs (`/contacts?orgId=...` for list, `/contacts` for create with `orgId` in body) hide the relationship from the URL. The hybrid is the readable middle.

**Tenant scoping.** Every endpoint resolves `actor.tenantId` from the JWT and filters/validates at the service layer. Cross-tenant reads return 404 (consistent with `CasesService.findOne`). Cross-tenant writes return 404 on the parent org lookup (a `POST /customers/:wrongTenantsOrgId/contacts` looks like the org doesn't exist to this actor).

**RBAC in P2.4.** Read endpoints are available to any authenticated ACTIVE user in the tenant. Write endpoints (POST + PATCH on both resources) are ADMIN-only — engineers gain write access in P2.7 once the engineer-facing UI surfaces a real need. The conservative default avoids accidentally exposing customer-data mutation to roles whose UX doesn't have it yet.

## Tests

**Backend:**

- `customers.service.spec.ts`:
  - `listOrgs` returns rows in tenant; status filter works; cross-tenant returns empty.
  - `findOrgById` returns the row; cross-tenant 404.
  - `createOrg` persists; default `status = ACTIVE` when omitted; rejects duplicate `(tenantId, name)` with conflict.
  - `updateOrg` partial-updates only present fields; cross-tenant 404; rejects rename to an existing name in the same tenant.
  - `listContactsForOrg` returns rows; cross-tenant returns empty.
  - `findContactById` returns the row; cross-tenant 404.
  - `createContact` writes with correct `tenantId` from parent (Pre-flight 11 invariant); rejects duplicate email under same org; cross-tenant parent 404.
  - `updateContact` partial-updates; email change re-checks unique-within-org; cross-tenant 404.
- `customers.controller.spec.ts` + `contacts.controller.spec.ts`:
  - Read endpoints accept any ACTIVE user.
  - Write endpoints reject non-admin with 403.
  - All endpoints pass `actor.tenantId` through to the service correctly.

**Frontend:**

- `customersSlice.test.ts`:
  - `fetchCustomersThunk` lazy-loads on first call; second call no-ops (RTK condition).
  - Selectors filter and sort correctly.
- `CustomerOrganizations.test.tsx`:
  - Empty state renders when no ACTIVE orgs.
  - Loading state renders during fetch.
  - Renders org rows with name + primary contact + contact count.
  - "View all" footer appears only when total > 10.
  - INACTIVE orgs are excluded from the widget.

## Acceptance criteria

- [ ] `GET /customers` returns ACTIVE + INACTIVE customer orgs in the actor's tenant (filter via `?status=` if present). `SafeUser`-style shape (no internal fields that don't exist in P2.4, but the pattern is set for future fields).
- [ ] `POST /customers` (ADMIN) creates an org with the supplied fields. `status` defaults to `ACTIVE` if not provided. Duplicate `(tenantId, name)` returns 409.
- [ ] `PATCH /customers/:id` (ADMIN) partial-updates the org. Status change from ACTIVE → INACTIVE works and is the canonical "remove from picker" path. INACTIVE → ACTIVE re-activation works.
- [ ] `POST /customers/:orgId/contacts` (ADMIN) creates a contact whose `tenantId` is set from the parent org's tenant — Pre-flight 11's cross-row invariant. Cross-tenant parent lookup returns 404.
- [ ] `PATCH /contacts/:id` (ADMIN) partial-updates. Email change re-checks the composite unique `(customerOrganizationId, email)`; collision returns 409.
- [ ] All write endpoints reject non-admin with 403.
- [ ] Cross-tenant isolation: every read returns 404 for rows in another tenant; every write rejects the parent lookup the same way.
- [ ] No DELETE endpoint exists in P2.4 (Pre-flight 13).
- [ ] Primary contact fields on `CustomerOrganization` are NOT auto-synced from `CustomerContact` writes (Pre-flight 14).
- [ ] Admin section at `/admin` renders the `CustomerOrganizations` widget alongside `PendingApprovalUsers` — no changes to `<AdminSection />`'s shell (the registry-accumulation proof).
- [ ] The widget shows ACTIVE orgs sorted by name, capped at 10 visible rows with a "view all" footer when more exist.
- [ ] `yarn workspace @caseflow-ai/backend test` clean.
- [ ] `yarn workspace @caseflow-ai/backend lint` clean.
- [ ] `yarn workspace @caseflow-ai/frontend test` clean.
- [ ] `yarn workspace @caseflow-ai/frontend lint` clean.
- [ ] `yarn workspace @caseflow-ai/frontend typecheck` clean.

## Resolved pre-flight decisions

The three pre-flights below should be drafted into `docs/decisions/phase-02-admin-and-personalization.md` alongside this spec and committed in the same change.

**Pre-flight 12 (P2) — Endpoint shape: nested for parent-context operations, flat for individual contact operations.**

- *Decision.* The customer-side REST surface is hybrid: parent-context operations on contacts use nested URLs (`POST /customers/:orgId/contacts`, `GET /customers/:orgId/contacts`); individual contact operations use flat URLs (`GET /contacts/:contactId`, `PATCH /contacts/:contactId`). Org operations are all flat under `/customers/:orgId` since there's no parent context above the org (the tenant is implicit via JWT).
- *Why hybrid.* The URL should express the relationship when the relationship is informative (creating a contact requires knowing which org it belongs to — nested URL makes that explicit) and stay quiet when the relationship is redundant (fetching contact by UUID doesn't need the parent org id since UUIDs are globally unique). The middle path also matches well-trodden REST conventions: Stripe, GitHub, and most mature REST APIs use this exact hybrid for parent/child resources.
- *Alternative rejected — fully-nested URLs (`/customers/:orgId/contacts/:contactId`).* Forces every contact endpoint to surface the parent context, including reads that don't need it. Worse, it duplicates information already encoded in the contact's UUID — a `PATCH /customers/wrong-org-id/contacts/correct-contact-id` is ambiguous: do we ignore the org id, or 404 on the mismatch? The flat individual endpoints sidestep this entirely.
- *Alternative rejected — fully-flat URLs (`/contacts?customerOrganizationId=...` for list, `/contacts` for create with org id in body).* Hides the relationship from the URL. A reader can't tell from the path that contacts are scoped to orgs. The create-by-body shape also makes the "this contact belongs to this parent org" check feel like a special case rather than the obvious URL constraint.
- *Revisit if.* P14 customer-app exposes a customer-facing contact-management surface where the customer hierarchy is different (e.g., contacts belong to multiple orgs through a junction table). At that point, the relationship is no longer "contact belongs to exactly one org" and the URL shape changes accordingly. No P2.4-level pressure to anticipate that.

**Pre-flight 13 (P2) — No DELETE endpoint in P2.4.**

- *Decision.* P2.4 ships no DELETE endpoint for either `CustomerOrganization` or `CustomerContact`. The canonical "we don't do business with them anymore" path is `PATCH /customers/:id { status: 'INACTIVE' }`. Contact removal is also deferred — if a contact is no longer relevant, the engineer-facing UI in P2.7 may surface a delete affordance gated behind confirmation, or the contact may simply be left in place. No production path requires hard delete in P2.4.
- *Why defer.* Two reasons.
  1. *Cases haven't FK'd to customer orgs yet.* P2.5 adds `Case.customerOrganizationId`. The right delete strategy depends on what that FK's `onDelete` constraint is, and that decision lives in P2.5's spec. Designing the delete endpoint before the FK exists invites a Mid-flight when P2.5 makes the call. Deferring is the cheaper move.
  2. *Status-INACTIVE covers the only real lifecycle.* No portfolio-demo scenario actually requires hard delete. The status enum handles "remove from picker without losing history" — which is the universal answer for SaaS customer records (real CRMs almost never hard-delete customer rows for the same reason).
- *Alternative rejected — Add DELETE in P2.4, refuse if cases exist.* Workable but premature. The case-FK exists only after P2.5; before then, the DELETE endpoint has no real check to perform beyond "does the org exist." Adding it now produces an endpoint that does the wrong thing for ~30 minutes of clock time during the P2.4 → P2.5 transition.
- *Alternative rejected — Add DELETE in P2.4 with CASCADE.* The schema has `ON DELETE CASCADE` from `customer_contacts` to `customer_organizations`, so a hard delete of an org would automatically drop its contacts. Acceptable in isolation, but loses correspondent history with no obvious recovery path. Inconsistent with the project's "never tombstone" stance on User rows.
- *Revisit if.* A real product need surfaces (e.g., GDPR right-to-erasure once P12 prod hardening puts compliance gates in scope). At that point, the DELETE story includes audit-log entries, case-FK cascade strategy, and probably a soft-delete `deletedAt` column. Today none of that infrastructure exists.

**Pre-flight 14 (P2) — Primary contact denormalized fields are NOT auto-synced from `CustomerContact` writes.**

- *Decision.* When a `CustomerContact` row is created or updated, the parent `CustomerOrganization`'s `primaryContactName` / `primaryContactEmail` / `primaryContactPhone` fields are NOT automatically modified. The denormalized fields are independent — set at org-create time (optionally auto-filled from the form's first contact entry), changed via explicit `PATCH /customers/:orgId` updates, and otherwise left alone. Pre-flight 10 (P2.3) accepted this drift as a feature, not a bug; P2.4 implements that explicitly by *not* writing a sync trigger.
- *Why no auto-sync.* Three reasons.
  1. *Pre-flight 10 already settled the architectural call.* The denormalized fields exist to serve a different purpose than the `customer_contacts` rows. Auto-syncing would conflate the two — once the org's "primary contact" line becomes a cached view of the first contact, the only-set-at-create-time semantics evaporate and we've effectively built an FK with extra steps.
  2. *Write amplification.* A contact update would have to propagate to potentially-stale org fields, requiring either a transaction or eventual-consistency machinery. Both are heavyweight for the actual product need.
  3. *Reader expectations stay clear.* The org's primary contact is the canonical at-a-glance summary, intentionally separate from the canonical correspondent list (`customer_contacts`). Anyone needing one or the other reads the right table. Mixing them via auto-sync breaks the mental model.
- *The "auto-fill from first contact" affordance.* The P2.7 create-org flow may offer a UI shortcut: when an engineer types a primary contact's name + email into the org form AND also adds them as the org's first contact, both surfaces should populate. That's a *one-shot fill at form submit*, not a runtime data binding. Same effect, fundamentally different semantics — and the right place for it is the UI layer, not the service layer.
- *Alternative rejected — Auto-sync via a service-layer trigger.* When `createContact` runs and the parent org has null primary fields, copy this contact's fields up. Sounds harmless but creates surprising behavior: the second contact added would also fail to populate (org already has primaries), creating an asymmetry that depends on insert order. Worse: deletes (if added later) would orphan the org's primary fields without a clear "promote a different contact" handoff.
- *Alternative rejected — Auto-sync only when fields are null.* Same first-write-wins shape, slightly less surprising. Still loses to the "if you want the org's primary contact to track a contact row, use an FK" objection.
- *Revisit if.* Pre-flight 10 is revisited (i.e., the denormalized fields are lifted into an FK + role table). At that point, the new model has explicit semantics for "this contact is the primary" and the question goes away. Until then, no sync.

## Out of scope (deferred)

- **Engineer-facing customer pages (list, detail, create / edit forms).** Lands in P2.7. P2.4 backend has the endpoints; P2.7 frontend wires the UX.
- **DELETE endpoints.** Pre-flight 13. Status flip via `PATCH` covers P2.4's needs.
- **Engineer write access.** Write endpoints in P2.4 are ADMIN-only. P2.7 may relax this for engineers once the UX justifies it (e.g., engineers correcting a customer contact's email mid-conversation).
- **Search / filtering UI on the admin widget.** Beyond the ACTIVE-first sort and the cap-at-10 truncation, no search. Engineer-facing pages in P2.7 pick up filtering + pagination.
- **Bulk operations.** No bulk-create contacts, no bulk-deactivate orgs. Single-row only.
- **Audit trail of customer mutations.** Who created the org, who last edited a contact, etc. Belongs in the future `audit_log` table (P3 sub-phase or later).
- **`Case.customerOrganizationId` FK + backfill.** Lands in P2.5. P2.4 customer rows exist standalone; P2.5 closes the loop with case linkage.
- **Customer-contact ↔ User linking.** Out of scope until P14 customer-app lands.
- **P13.3 email-in sender matching against `CustomerContact.email`.** Read pattern only — P2.4 surfaces the data; P13 implements the match.
- **Per-tenant default contact / branding templates.** Out of scope.

## How we'll work this sub-phase

Three chunks with review checkpoints between each. Same pattern as P1.7, P1.8, P2.1, P2.2, P2.3.

### Chunk A — Backend CRUD + tests

1. `backend/src/customers/dto/` — five DTOs as listed under "Affected areas."
2. `backend/src/customers/customers.service.ts` — eight methods, with the cross-tenant FK invariant enforced on `createContact` + `updateContact`.
3. `backend/src/customers/customers.controller.ts` — REST controller for `/customers` and `/customers/:orgId/contacts`.
4. `backend/src/customers/contacts.controller.ts` — flat controller for `/contacts/:id`.
5. `backend/src/customers/customers.module.ts` — register service + both controllers; export service for future case-FK validation.
6. Three spec files: `customers.service.spec.ts`, `customers.controller.spec.ts`, `contacts.controller.spec.ts`.
7. Smoke via curl + `dev.sh db:psql` that the endpoints behave: create an org under Tenant A as admin; verify the seed-data engineer in Tenant A can read it; verify a Tenant B admin gets 404; create a contact and verify `tenantId` is set from the parent; verify a duplicate-email-under-same-org is rejected.

**Review checkpoint 1.** Backend endpoints behave per the API-surface table. Tenant isolation verified end-to-end. Cross-tenant FK invariant enforced.

### Chunk B — Frontend widget + slice + tests

1. `frontend/src/features/customers/` — `customersApi.ts`, `customersSlice.ts`, `types.ts`, `index.ts`, slice tests.
2. `frontend/src/features/admin/widgets/CustomerOrganizations.tsx` — the widget, with empty / loading / rows / "view all" states.
3. `frontend/src/features/admin/widgets/CustomerOrganizations.test.tsx` — render tests.
4. `frontend/src/features/admin/registry.ts` — append the widget.
5. `frontend/src/store/index.ts` — register the `customers` reducer.
6. Manual smoke: log in as a Tenant 1 admin; verify the `/admin` page shows both widgets; verify seeded customer orgs appear correctly sorted (ACTIVE first), `INACTIVE` excluded, contact-count chip accurate.

**Review checkpoint 2.** Admin section now has two widgets. The page shell (`<AdminSection />`) has not changed — the registry pattern's accumulation payoff is visible end-to-end.

### Chunk C — Documentation + decision-log pre-flights + ROADMAP flip

1. Draft Pre-flights 12–14 in `docs/decisions/phase-02-admin-and-personalization.md` (bodies stubbed in this spec under "Resolved pre-flight decisions"). They join the existing Pre-flights 1–11 + Mid-flights 1–5.
2. Update `frontend/src/features/admin/README.md` (if it exists from P2.1) with the new widget; otherwise note that the registry now has two widgets and link to both implementations.
3. ROADMAP — flip P2.4 from `⬜` to `✅` with the implementation date. Advance the P2 By-phase row to `█████░░░░░ 57% 4/7`.
4. ROADMAP — refresh "Current state" block: shipped date, P2 progress, what's next (P2.5 — Case FK + backfill).
5. Showcase repo sync (per the ongoing-updates directive): copy the updated ROADMAP, the updated decision log, and this spec into `caseflow-ai-showcase/`.

**Review checkpoint 3.** Pre-flights captured. ROADMAP flipped. Showcase repo synced. P2.4 close-out complete.

## Risks and how we'll handle them

- **Cross-tenant FK invariant slip.** The DB cannot enforce that `customer_contacts.tenantId` matches its parent org's `tenantId`. P2.4's `CustomersService.createContact` + `updateContact` are the only place this is enforced. Mitigation: explicit unit tests assert the rejection path with a hostile request shape (`POST /customers/:tenantBsOrgId/contacts` from a Tenant A admin → 404 because the parent org lookup is tenant-scoped, so the actor can't even reach the orgId). The test names this exact case.
- **Status enum vs. soft-delete confusion.** Engineers reviewing the code might expect a `deletedAt` column or DELETE endpoint and not find one. Mitigation: this Pre-flight 13 is the explicit answer; the controller has no DELETE handler; the entity has no `deletedAt`. The decision-log entry covers the rationale.
- **Widget vs. engineer-page scope confusion.** The widget being read-only might look incomplete during P2.4 review. Mitigation: this spec's "Goal" section calls out that the engineer-facing CRUD UI lives in P2.7 explicitly; the widget's role is composition proof. Plus the "Open architectural calls" history (P2.1 Pre-flight 2) already established this widget as the test case for registry accumulation.
- **Optimistic update on status flips.** Flipping an org `ACTIVE → INACTIVE` from the widget would remove it from the visible list. P2.4 widget doesn't expose status-flip actions (no edit UI in the widget); the slice will surface the action method for P2.7 to consume. Mitigation: P2.4 ships the slice's `updateOrgStatusThunk` with optimistic update + rollback pattern (mirroring `updateUserStatusThunk` from P2.1) but no UI consumer yet. P2.7 wires it in. Documented in the slice's docstring.
- **Existing duplicate-prevention via composite unique.** The `(tenantId, name)` UNIQUE on `customer_organizations` and `(customerOrganizationId, email)` UNIQUE on `customer_contacts` will throw on conflict — TypeORM's default error is a `QueryFailedError` with `code: '23505'`. Mitigation: the service catches this and translates to a `ConflictException` with a useful message. Both happy-path and conflict-path tests cover this.
- **Email casing on contact create + update.** Same pattern as `users.email` — service layer lowercases before persist. Mitigation: explicit test confirms case-insensitive uniqueness (creating "JANE@example.com" then "jane@example.com" under the same org returns 409, not two rows).
- **Widget render on tenants with zero customer orgs.** The default tenant (`abc-company`) has 3 seeded orgs; a freshly-created tenant has zero. Empty state must render cleanly. Mitigation: widget test explicitly covers the zero-rows case; manual smoke step creates a new tenant via dev-only helper and verifies.
- **RBAC inconsistency between read and write endpoints.** Read endpoints accept any ACTIVE user; write endpoints require ADMIN. A potential gotcha is the existing pattern in P1.6 where `<RequireRole>` is route-level — P2.4's RBAC is per-endpoint and lives in the controller. Mitigation: explicit `@CurrentUser()` role check inside each handler, mirroring `UsersController.updateStatus` from P2.2. Spec tests cover the 403 path for non-admins on every write endpoint.

## Followup

After P2.4 ships:

- **P2.5 (`Case.customerOrganizationId` FK + backfill)** is the next sub-phase. P2.4 establishes the customer-org rows that P2.5 will FK into. The backfill picks one org per tenant (e.g., the first ACTIVE org from the seed) to assign to existing P1-era cases.
- **P2.7 (engineer-facing customer pages)** consumes the rest of the P2.4 endpoint surface — create form, edit form, contact management — plus the slice's `updateOrgStatusThunk`. The widget stays read-only; the engineer page is the full CRUD surface.
- **DELETE endpoints** revisit after P2.5 lands and the case-FK story is settled (Pre-flight 13's revisit trigger).
- **Audit trail** — when the project introduces an `audit_log` table (P3 or later), customer mutations are natural early candidates ("who deactivated this org?").
- **P13.3 sender matching** depends on stable `customer_contacts.email` rows being creatable via the API. P2.4 closes that gap.
- **ROADMAP P2 status flips to `█████░░░░░ 57% 4/7` after P2.4.**
- **Content series.** This sub-phase is a strong demo of "two widgets, one shell, one registry" — natural article anchor for the **Article 11 — Dev-ready specs for AI pair programming** when the article-series schedule reaches it.

## Confidence

**Confidence:** 94%. The CRUD endpoints are mechanical against well-defined entities. The cross-tenant FK invariant is the one place runtime logic matters; it's bounded to two service methods with explicit test coverage. Pre-flights 12–14 are all small calls with clear revisit triggers. The widget reuses the P2.1 patterns end-to-end (slice + thunk + memoized selectors + admin registry). The only architectural surprise risk is the no-DELETE call — explicitly captured in Pre-flight 13 so future contributors find the rationale before re-implementing it.
