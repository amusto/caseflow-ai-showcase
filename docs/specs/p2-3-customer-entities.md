# Dev-Ready Task — P2.3 · CustomerOrganization + CustomerContact entities

**Phase:** [P2](../../../../ROADMAP.md#phase-2--admin-section--customer-model-) · Sub-phase **P2.3**
**Decision log:** [`docs/decisions/phase-02-admin-and-personalization.md`](../../../../docs/decisions/phase-02-admin-and-personalization.md) — Pre-flights 9–11 (drafted alongside this spec; see "Resolved pre-flight decisions" below)
**Status:** dev-ready

## Goal

Open the customer-model theme of Phase 2 by landing the two foundational entities — `CustomerOrganization` and `CustomerContact` — plus a new `CustomersModule` to house them. No CRUD endpoints, no DTOs, no controller, no frontend; pure schema + module scaffold + entity tests + demo-seed extension.

P2.3 is the same shape as P1.1 was for `Tenant`: a one-sub-phase commitment to the data model that subsequent sub-phases consume. P2.4 builds the CRUD service + controller on top of these entities; P2.5 adds the `customerOrganizationId` FK to `Case`; P2.7 builds the engineer-facing customer pages.

Keeping P2.3 pure-schema is a deliberate scoping choice. Schema is the one decision that's expensive to walk back once data lands — splitting it from the CRUD work lets the architectural calls in Pre-flights 9–11 get reviewed independently of the (mechanical) endpoint work that follows.

## Why this architecture (the 90-second mental model)

Two entities, one module, two tenant boundaries.

**`CustomerOrganization`** is the company / account that owns one or more cases (the FK from `Case` lands in P2.5). It's tenant-scoped — every customer org belongs to exactly one tenant, and no query crosses tenants. The row carries enough denormalized contact information (`primaryContactName`, `primaryContactEmail`) to power an at-a-glance list view without joining to `customer_contacts` — see Pre-flight 10 for why denormalization wins here.

**`CustomerContact`** is the individual person at a customer org we correspond with. Multiple contacts per org. Tenant-scoped via a denormalized `tenantId` column (Pre-flight 11 — same pattern as `cases`, `users`), AND customer-org-scoped via the `customerOrganizationId` FK. Email is unique within the customer org (Pre-flight 11; composite unique index on `(customerOrganizationId, email)`) — the same person at two different companies is two `CustomerContact` rows, not one.

**One `CustomersModule`** houses both entities (Pre-flight 9). Splitting into `CustomerOrganizationsModule` + `CustomerContactsModule` creates artificial boundaries between two tables that are joined on nearly every query. The single module pattern matches `UsersModule` (which houses `User` + future user-related entities) and `CasesModule` (which houses `Case` + `CaseHistory`).

**Status, not soft-delete.** `CustomerOrganization.status` is an enum (`ACTIVE` / `INACTIVE`) — the inactive state hides the org from case-creation pickers (P2.7) without losing historical cases. Same shape as `User.status` from P1.2 — readable, reversible, never tombstone.

**No `synchronize: true` migration scripts.** Schema lands through TypeORM's `synchronize` per the P1.1 / Mid-flight 1 reversal. The new tables (`customer_organizations`, `customer_contacts`) appear after the first backend boot under the merged feature branch. Manual migration files arrive at P12 prod hardening with the rest of the explicit-migration discipline.

**What's deliberately NOT here.**

- No CRUD endpoints. `CustomersController` + `CustomersService` land in P2.4. The empty module ships only the entities and the `TypeOrmModule.forFeature([CustomerOrganization, CustomerContact])` registration.
- No DTOs. They land alongside the controller in P2.4.
- No frontend changes. Engineer-facing surfaces (list, detail) land in P2.7.
- No `Case.customerOrganizationId` FK. Lands in P2.5 with the backfill migration.
- No `CustomerContact` ↔ `Case` linkage (e.g., per-case correspondence). Pure customer-model layer here; case linkage is its own concern.

## Affected areas

### Backend — new files

**Customers feature folder (`backend/src/customers/`):**

- `customers.module.ts` — registers the module. Imports `TypeOrmModule.forFeature([CustomerOrganization, CustomerContact])` and `TenantsModule` (for the FK target, although neither entity needs a service-layer call to it in P2.3). Providers + controllers are empty arrays in P2.3 — added in P2.4.
- `entities/customer-organization.entity.ts` — `@Entity({ name: 'customer_organizations' })` with the columns described under "Schema."
- `entities/customer-contact.entity.ts` — `@Entity({ name: 'customer_contacts' })` with the columns described under "Schema."
- `entities/customer-organization.entity.spec.ts` — entity smoke test mirroring `tenants/entities/tenant.entity.spec.ts`. Verifies the entity metadata (table name, column types, indexes) loads cleanly under TypeORM.
- `entities/customer-contact.entity.spec.ts` — same shape.

**Enum (`backend/src/common/enums/`):**

- `customer-organization-status.enum.ts` — `enum CustomerOrganizationStatus { ACTIVE = 'ACTIVE', INACTIVE = 'INACTIVE' }`. String-valued like the other enums in `common/enums/` (consistent with the postgres column type `enum` storage).

### Backend — modified files

- `backend/src/app.module.ts` — register `CustomersModule` alphabetically alongside the other feature modules (between `CasesModule` and `NotificationsModule`).
- `backend/src/database.config.ts` — no change. `entities` glob already picks up the new entity files automatically.

### Demo seed harness (P1.9 follow-up)

- `backend/src/seed/factories/customer-organizations.factory.ts` — new factory mirroring the P1.9 pattern (`tenants.factory.ts`, `users.factory.ts`, `cases.factory.ts`). Produces 2–3 customer orgs per tenant with deterministic UUIDs via `seed/util/deterministic-uuid.ts`.
- `backend/src/seed/factories/customer-contacts.factory.ts` — same pattern, 2–3 contacts per org.
- `backend/src/seed/seed.service.ts` — extend `runFor(tenant)` (or its equivalent) to invoke the new factories after the existing users/cases seed pass. Re-running `dev.sh demo:seed` stays idempotent because the deterministic UUIDs make `repo.save(...)` upsert on conflict (same property P1.9's existing seed relies on).

The seed extension is part of P2.3 (not deferred to P2.4) because:

- P1.9 established `dev.sh demo:reset && dev.sh demo:seed` as the canonical "fresh state" command, and that command should never produce a half-populated schema. Adding columns without seeding them leaves the demo in an awkward in-between.
- The seed data answers a real question during P2.4 review: "what shape should the list view actually look like?" Seeded rows make that conversation concrete.

### Tests

- `customer-organization.entity.spec.ts` — entity metadata smoke test (columns, indexes, table name).
- `customer-contact.entity.spec.ts` — entity metadata smoke test, plus the composite unique index assertion (`(customerOrganizationId, email)`).
- Seed harness test (extension of the existing P1.9 seed test if one exists; otherwise a new file) — verifies the seeded customer-org and contact counts per tenant match the expected fixture.

No service or controller tests in P2.3. Those land in P2.4 with the CRUD work.

## Schema

### `customer_organizations`

| Column                | Type                                                 | Notes                                                                                |
| --------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `id`                  | `uuid` PK                                            | `@PrimaryGeneratedColumn('uuid')`                                                    |
| `tenantId`            | `uuid` NOT NULL                                      | FK → `tenants(id)` `ON DELETE RESTRICT`. Indexed (`ix_customer_organizations_tenant_id`). |
| `name`                | `varchar(160)` NOT NULL                              | Display name. Same max length as `tenants.name`.                                     |
| `primaryContactName`  | `varchar(120)` NULL                                  | Denormalized — see Pre-flight 10. Optional because a customer org may be created before contacts are entered. |
| `primaryContactEmail` | `varchar(254)` NULL                                  | Denormalized — see Pre-flight 10. RFC 5321 max-length for email.                     |
| `primaryContactPhone` | `varchar(32)` NULL                                   | Free-text phone — no E.164 validation in P2.3 (see "Out of scope").                  |
| `status`              | `enum CustomerOrganizationStatus` NOT NULL `default 'ACTIVE'` | Drives case-creation picker visibility in P2.7.                          |
| `createdAt`           | `timestamp` NOT NULL                                 | `@CreateDateColumn`                                                                  |
| `updatedAt`           | `timestamp` NOT NULL                                 | `@UpdateDateColumn`                                                                  |

Indexes beyond the PK:

- `ix_customer_organizations_tenant_id` on `(tenantId)` — supports the tenant-scoped list query in P2.4.
- `ux_customer_organizations_tenant_name` UNIQUE on `(tenantId, name)` — a customer org's name must be unique within its tenant. Two tenants can both have a customer called "Acme"; one tenant can't have two.

### `customer_contacts`

| Column                    | Type                       | Notes                                                                                |
| ------------------------- | -------------------------- | ------------------------------------------------------------------------------------ |
| `id`                      | `uuid` PK                  | `@PrimaryGeneratedColumn('uuid')`                                                    |
| `tenantId`                | `uuid` NOT NULL            | FK → `tenants(id)` `ON DELETE RESTRICT`. Denormalized for tenant-scoped queries (Pre-flight 11). Indexed (`ix_customer_contacts_tenant_id`). |
| `customerOrganizationId`  | `uuid` NOT NULL            | FK → `customer_organizations(id)` `ON DELETE CASCADE` — delete the org, drop its contacts. Indexed (`ix_customer_contacts_org_id`). |
| `name`                    | `varchar(120)` NOT NULL    |                                                                                      |
| `email`                   | `varchar(254)` NOT NULL    | RFC 5321 max length. Lowercased on write (P2.4 service-layer concern).               |
| `phone`                   | `varchar(32)` NULL         | Free-text.                                                                           |
| `createdAt`               | `timestamp` NOT NULL       | `@CreateDateColumn`                                                                  |
| `updatedAt`               | `timestamp` NOT NULL       | `@UpdateDateColumn`                                                                  |

Indexes beyond the PK:

- `ix_customer_contacts_tenant_id` on `(tenantId)` — tenant-scoped queries.
- `ix_customer_contacts_org_id` on `(customerOrganizationId)` — parent-child queries.
- `ux_customer_contacts_org_email` UNIQUE on `(customerOrganizationId, email)` — see Pre-flight 11. Same email can exist under two different orgs; not under the same org twice.

## Acceptance criteria

- [ ] `customer_organizations` and `customer_contacts` tables exist in the dev database after `synchronize` runs on backend boot.
- [ ] Both tables carry the columns, indexes, and FK constraints described under "Schema."
- [ ] `CustomersModule` registers cleanly in `AppModule`; backend boots without errors.
- [ ] `customer-organization.entity.spec.ts` and `customer-contact.entity.spec.ts` pass — entity metadata loads, columns are typed correctly, indexes are declared as expected.
- [ ] `dev.sh demo:reset && dev.sh demo:seed` produces 2–3 customer organizations per tenant and 2–3 contacts per organization, all with deterministic ids. Re-running the seed is idempotent.
- [ ] Cross-tenant FK enforcement: inserting a `customer_contact` whose `tenantId` doesn't match its parent `customer_organization.tenantId` is caught at the application layer (P2.4 service-layer assertion; P2.3 documents the invariant in code comments). The DB doesn't enforce this cross-row consistency natively — the service layer must, and the comment on the entity makes that explicit.
- [ ] No frontend changes.
- [ ] No new HTTP routes.
- [ ] `yarn workspace @caseflow-ai/backend test` clean.
- [ ] `yarn workspace @caseflow-ai/backend lint` clean.
- [ ] No proxy / CloudFront behavior changes.

## Resolved pre-flight decisions

The three pre-flights below should be drafted into `docs/decisions/phase-02-admin-and-personalization.md` alongside this spec and committed in the same change.

**Pre-flight 9 (P2) — Customer module structure: one `CustomersModule` housing both entities.**

- *Decision.* `CustomerOrganization` and `CustomerContact` live in a single feature module at `backend/src/customers/`. The module's `TypeOrmModule.forFeature([CustomerOrganization, CustomerContact])` registers both; P2.4's `CustomersService` + `CustomersController` will own both entities' CRUD; P2.7's frontend `features/customers/` mirrors the same coupling.
- *Alternative rejected — split into `CustomerOrganizationsModule` + `CustomerContactsModule`.* Tempting on "one module per entity" grounds, but the two entities are parent/child and joined on nearly every query of interest ("list contacts for this org," "list orgs with their primary contact," etc.). Splitting forces either (a) a circular import between two modules that need each other's services, or (b) a third "shared" module that holds both — which is the single-module decision with extra steps. Single module is the cleaner shape.
- *Alternative rejected — fold into `UsersModule`.* `User` and `CustomerContact` are both "people-ish" rows, but they answer different questions (authentication subject vs. customer-side correspondent) and have different lifecycle, retention, and PII concerns. Folding them invites conflation. Separate modules with clear boundaries beats one module with a mixed concept.
- *Revisit if.* Either entity grows enough sub-concepts (e.g., `CustomerOrganization` accumulating contracts, billing addresses, tags) that splitting eases reasoning. The split is purely additive at that point — the existing entities don't move; new sub-entities land in new modules.

**Pre-flight 10 (P2) — Primary contact: denormalized fields on `CustomerOrganization`, not an FK to `CustomerContact`.**

- *Decision.* `CustomerOrganization` carries `primaryContactName`, `primaryContactEmail`, and `primaryContactPhone` directly as nullable columns. There is no `primaryContactId` FK pointing into `customer_contacts` in P2.3. The denormalized fields can be populated from any of: a user-typed value at org-create time (most common in P2.4's UI), an auto-fill from the first `CustomerContact` row, or an explicit "set as primary" action on a contact (P2.7-ish).
- *Why denormalize.* Three reasons.
  1. **Chicken-and-egg.** A customer org can plausibly exist before any individual contact is entered (e.g., a sales handoff: "we just signed Acme; details to follow"). Requiring a `CustomerContact` row before the org can be saved adds awkward two-step creation flows or forces stub contacts that are worse than denormalized strings.
  2. **List-view performance.** The admin and engineer list views show the org name and primary contact at a glance. Denormalization makes that a single-table scan; the FK alternative requires a left join on every list query. At P2.4's scale, the join is cheap — at P11's scale, denormalization stays cheap.
  3. **Edit ergonomics.** Updating "the primary contact's email" via a contact row creates a write-amplification expectation (write to `customer_contacts`, propagate to `customer_organizations.primaryContactEmail`). Denormalization makes the two writes independent and lets the org's "primary contact" line carry a freely-typed value when the contact isn't a system-of-record entry yet.
- *Cost of the decision.* Drift risk: if a `CustomerContact` row's email changes, the `CustomerOrganization.primaryContactEmail` may go stale. Mitigation: P2.4's update flow exposes both as separately-editable fields. The "primary contact" line on the org is the canonical at-a-glance summary; the `customer_contacts` rows are the canonical correspondent list. They serve different purposes; drift between them is acceptable.
- *Alternative rejected — `primaryContactId` UUID FK into `customer_contacts`.* Defers the at-a-glance display problem to a join, and creates the chicken-and-egg constraint above. Cleaner normalization at the cost of every consumer paying the join tax and the create-flow complexity.
- *Alternative rejected — JSON column with the full contact blob.* Considered briefly. Loses the type safety of typed columns and the indexability of `primaryContactEmail` (which P13's email-in sender matching will eventually want — see ROADMAP P13.3, where `CustomerContact.email` is the primary match, but the org's primary contact email is a useful fallback).
- *Revisit if.* The "primary contact" notion grows from a single denormalized line into a richer concept (e.g., "primary billing contact," "primary technical contact," "primary escalation contact"). At that point, lifting into a proper FK + role table is the right move, and the denormalized columns either go away or become a cached "default contact" view.

**Pre-flight 11 (P2) — `CustomerContact` email uniqueness: scoped to the customer organization, not the tenant or global.**

- *Decision.* `customer_contacts.email` is uniqueness-scoped via a composite unique index on `(customerOrganizationId, email)`. Two different customer orgs can each have a contact with the same email address (legitimate: a consultant working with multiple client orgs). One customer org cannot have the same email listed twice.
- *Why not global.* A global unique index on email would prevent the consultant case above. It would also create cross-tenant collision potential — Tenant A's customer's email collides with Tenant B's customer's email, which the system would refuse to insert. That's wrong: the two tenants don't share a customer namespace, and a customer's email isn't a system-level identifier.
- *Why not tenant-scoped.* A tenant-scoped unique on `(tenantId, email)` would prevent one of your tenant's customer orgs from listing a contact that also exists at another of the same tenant's customer orgs. Defensible but restrictive: a consulting tenant managing multiple end-client orgs would constantly hit this collision for shared contacts.
- *Why org-scoped is the right grain.* It matches how contacts are mentally used: "this person represents Acme in dealings with us." Two Acme records with the same email is a duplicate-row bug; the same email at Acme and at Globex is two correspondents in two relationships.
- *Tenant scoping (separately).* Tenant scope is enforced via the denormalized `tenantId` column + the `(tenantId)` index on `customer_contacts`. The composite unique on `(customerOrganizationId, email)` is the uniqueness rule; the tenant index is the query-perf rule. They serve different purposes and don't conflict.
- *Email casing.* Stored lowercased (P2.4 service-layer concern; same pattern as `users.email`). The unique index is therefore effectively case-insensitive without needing `LOWER(email)` expression indexes.
- *Alternative rejected — global unique on `email`.* See above. Wrong scope.
- *Alternative rejected — tenant-scoped unique `(tenantId, email)`.* See above. Defensible but restrictive.
- *Revisit if.* The email-in surface (P13.3) needs to match an inbound email to exactly one `CustomerContact` and the org-scoped unique allows ambiguity (the same address appears under two orgs in the same tenant). At that point, the match algorithm becomes "by email + most-recently-corresponded org" or similar; the index doesn't change but the resolution rule grows.

## Out of scope (deferred)

- **CRUD endpoints, DTOs, service.** Lands in P2.4. P2.3 ships entities + module shell only.
- **`Case.customerOrganizationId` FK.** Lands in P2.5 with the backfill migration for P1-era seeded cases. P2.3 doesn't touch `cases`.
- **Engineer-facing customer pages.** Lands in P2.7 (list page, detail page, case-creation form requiring an org pick).
- **Customer-organization admin widget.** Lands in P2.4 as the second widget on the admin section (proves the widget-registry pattern accumulates without page-shell changes — see Pre-flight 2).
- **Phone number validation / E.164 normalization.** Phones are free-text `varchar(32)`. Adding `libphonenumber-js` or similar is its own decision and adds a dependency that isn't earned at the schema layer.
- **Address fields on customer organization.** Customer billing/shipping addresses don't exist yet. Add when a real consumer asks (likely P14 customer-app or P12 prod hardening's invoicing surface).
- **Customer-organization tags / custom fields.** Tagging or per-tenant custom attributes (e.g., "industry," "tier") are a stretch feature. The schema stays focused on the universally-needed columns; extensions land in their own sub-phases.
- **Soft delete for customer organizations.** Status enum (`ACTIVE` / `INACTIVE`) covers the lifecycle need without a soft-delete column. Archival semantics, if ever needed, can grow on top of status without a schema migration.
- **Per-tenant default contact templates.** A tenant might want every new customer to default to certain primary-contact conventions. Out of scope; revisit if observed in real demo usage.
- **Customer-contact ↔ user linking.** If a `CustomerContact` is also a `User` of the system (e.g., a customer-portal user from P14), there's no FK linking them in P2.3. The two concepts are tracked separately until P14 lands the customer-portal surface and the linkage decision becomes concrete.
- **Migration files.** Per the P1.1 / Mid-flight 1 reversal, dev uses `synchronize: true`. Explicit migration files arrive at P12 prod hardening.

## How we'll work this sub-phase

Three chunks with review checkpoints between each. Same pattern as P1.1, P1.7, P1.8, P2.1, P2.2.

### Chunk A — Entities + module + tests

1. `backend/src/common/enums/customer-organization-status.enum.ts` — new enum.
2. `backend/src/customers/entities/customer-organization.entity.ts` — entity class.
3. `backend/src/customers/entities/customer-contact.entity.ts` — entity class.
4. `backend/src/customers/customers.module.ts` — module shell. Imports `TypeOrmModule.forFeature([...])` and `TenantsModule`. Empty `providers` and `controllers` (P2.4 fills them).
5. `backend/src/app.module.ts` — register `CustomersModule`.
6. `entities/customer-organization.entity.spec.ts` and `entities/customer-contact.entity.spec.ts` — entity smoke tests mirroring `tenants/entities/tenant.entity.spec.ts`. Verify table name, columns, indexes, and FK target.
7. Boot the backend locally; confirm the two tables appear under `synchronize` (`\dt` in psql).
8. `yarn workspace @caseflow-ai/backend lint` clean; `yarn workspace @caseflow-ai/backend test` clean.

**Review checkpoint 1.** Tables exist with the expected shape. Entity tests pass. Module registers without errors. The `dev.sh db:psql` `\dt` output shows `customer_organizations` and `customer_contacts` alongside the existing P1-era tables.

### Chunk B — Demo seed extension

1. Add `backend/src/seed/factories/customer-organizations.factory.ts` and `customer-contacts.factory.ts` following the existing `tenants.factory.ts` / `users.factory.ts` / `cases.factory.ts` pattern. Deterministic UUIDs via `seed/util/deterministic-uuid.ts` so re-running is idempotent.
2. Mix of statuses: at least one `INACTIVE` customer org per tenant so the P2.7 case-creation picker filter logic has a real "this one should be hidden" case to test against later.
3. Vary contact counts: one org has the minimum (1 contact = the primary); another has 2-3 contacts. The variation gives P2.4's UI work realistic data.
4. Smoke: `dev.sh demo:reset && dev.sh demo:seed`; `\d+ customer_organizations` and `SELECT name, status FROM customer_organizations` in psql confirm the rows.
5. If a seed test exists from P1.9, extend it with row-count assertions for the new tables.

**Review checkpoint 2.** Seed produces deterministic, idempotent customer-side data. `INACTIVE` rows present. Row counts match the expected fixture.

### Chunk C — Documentation + decision-log pre-flights + ROADMAP flip

1. Draft Pre-flights 9–11 in `docs/decisions/phase-02-admin-and-personalization.md` (bodies stubbed in this spec under "Resolved pre-flight decisions"). They join the existing Pre-flights 1–8 and Mid-flights 1–4.
2. ROADMAP — flip P2.3 from `⬜` to `✅` with the implementation date. Advance the P2 By-phase row to `███░░░░░░░ 43% 3/7`.
3. ROADMAP — refresh "Current state" block: shipped date, P2 progress, what's next (P2.4).

**Review checkpoint 3.** Pre-flights captured. ROADMAP flipped. Decision log link from this spec resolves.

## Risks and how we'll handle them

- **Cross-tenant FK consistency on `customer_contacts`.** The denormalized `tenantId` on `customer_contacts` must match its parent `customer_organizations.tenantId`. The DB doesn't enforce cross-row consistency natively — it could allow inserting a contact under Tenant A's org with `tenantId = Tenant B`. Mitigation: the entity file carries an explicit comment documenting the invariant; P2.4's `CustomersService.createContact` enforces it by reading the parent org's tenantId and rejecting any mismatch. P2.6 (tests sub-phase) explicitly tests this invariant. The denormalized column is a query-perf optimization, not the source of truth — `customer_organizations.tenantId` is.
- **Schema drift from `synchronize` between dev branches.** Two engineers (or two parallel feature branches) adding overlapping columns could produce inconsistent dev schemas. Mitigation: the P1.1 decision-log entry (Mid-flight 1 reversal) explicitly accepts this risk for dev; the resolution path is `dev.sh demo:reset` followed by a re-seed. At P12 prod hardening, explicit migrations replace `synchronize` and this risk goes away.
- **`primaryContactEmail` drift from the matching `CustomerContact.email`.** Pre-flight 10 accepts this as a feature (the two fields serve different purposes), but a future contributor might assume they're meant to track. Mitigation: explicit code comment on the `primaryContactEmail` column declaring its independence from the contacts table; P2.4 review checks that update flows expose both as separately-editable.
- **Existing seeded cases (P1.9) now have no `customerOrganizationId` because the column doesn't exist yet.** P2.5 lands that column and the backfill. P2.3's seed extension introduces customer-side data that P2.5's backfill will assign existing cases to. No P2.3 risk; just a note that the case data is correctly orphan-of-customer until P2.5 closes the loop.
- **`ON DELETE CASCADE` on `customer_contacts.customerOrganizationId`.** Deleting a customer org drops all its contacts. P2.4's delete flow will likely gate this behind a confirmation modal; P2.3 sets the schema constraint. Mitigation: explicit comment on the entity declaring why CASCADE (vs. RESTRICT) — contacts are children, not independent rows; orphan contacts are nonsense.
- **`ON DELETE RESTRICT` on `customer_organizations.tenantId` and `customer_contacts.tenantId`.** Matches existing project pattern (`cases.tenantId`, `users.tenantId`). Tenant deletion is not a flow this project exposes; if it ever does (P12 prod hardening?), the cascade strategy is its own decision. Mitigation: documented in the entity comment.
- **PII concentration.** Customer-contact rows hold name + email + phone — meaningful PII. Mitigation: the `docs/planning/soc2-readiness.md` doc should pick up a one-line entry noting that the customer-contact surface is new PII storage; the data-retention + access-control decisions on this surface land at P12 prod hardening. No P2.3-blocking action.

## Followup

After P2.3 ships:

- **P2.4 (Customer CRUD + admin widget)** is the next sub-phase. With entities in place, the CRUD work is mechanical: DTOs + service + controller + the `CustomerOrganizations` admin widget. Decision log shouldn't need new pre-flights for P2.4 — the architectural calls are all in P2.3 (modeling) and P2.1 (admin composition).
- **P2.5 (`Case.customerOrganizationId` FK + backfill)** follows. The seeded customer data from P2.3 is what the backfill assigns existing cases to.
- **P13.3 (email-in sender matching)** depends on `CustomerContact.email` being uniquely resolvable to a single contact. The P2.3 schema (`(customerOrganizationId, email)` unique, plus the per-org match in P13.3) gives that primary path; the fallback case ("email matches contacts in two different orgs in the same tenant") is acknowledged in Pre-flight 11's "Revisit if."
- **`docs/planning/soc2-readiness.md`** — add a one-line entry under PII-touched surfaces noting that `customer_contacts` is a new PII storage location; full retention + access-control treatment lands with the rest of P12 prod hardening.
- **ROADMAP P2 status flips to `███░░░░░░░ 43% 3/7` after P2.3.**

## Confidence

**Confidence:** 97%. The architectural calls are all small (module structure, denormalization, uniqueness scope), each with a clearly-bounded revisit trigger. The schema is conservative — universal columns only, no speculative fields. The seed extension is mechanical pattern-match against P1.9. The one real risk (cross-tenant FK consistency on `customer_contacts`) is mitigated by documenting the invariant in P2.3 and enforcing it in P2.4's service layer — a deliberate split that keeps P2.3 pure-schema and lets the enforcement work get reviewed alongside its service-layer logic.
