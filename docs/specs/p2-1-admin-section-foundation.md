# Dev-Ready Task — P2.1 · Admin Section Foundation

**Phase:** [P2](../../../../ROADMAP.md#phase-2--admin-section--customer-model-) · Sub-phase **P2.1**
**Decision log:** [`docs/decisions/phase-02-admin-and-personalization.md`](../../../../docs/decisions/phase-02-admin-and-personalization.md) — Pre-flight 2 (drafted alongside this spec; see "Resolved pre-flight decisions" below)
**Status:** dev-ready

## Goal

Stand up a new `/admin` route surface — gated to `role === 'ADMIN'` — and reuse the widget-registry composition primitive from P1.8 to host admin-facing widgets. Ship the first widget: **PendingApprovalUsers**, which lists users in `PENDING_APPROVAL` status for the current tenant and exposes one-click approve / reject buttons backed by the existing `PATCH /users/:id/status` endpoint.

P2.1 is foundation. The widget set is intentionally one, not three — the architectural payoff is that subsequent admin features (P2.4 customer organizations, P2.7+ case templates, etc.) drop into the existing registry without touching the page shell. Same payoff structure as P1.8 → P1.9+, just on the admin surface.

Engineers continue to see the engineer dashboard at `/`. ADMIN users see both surfaces — engineer dashboard at `/` and admin section at `/admin` — connected by a nav link in the app bar that appears only when their role is `ADMIN`.

## Why this architecture (the 90-second mental model)

The pattern from P1.8 carries over verbatim. The whole admin section hinges on the same `Widget` contract:

```ts
interface Widget {
  id: string;                          // 'admin.pending-approval-users'
  title: string;                       // 'Pending approval'
  audience: UserRole[];                // ['ADMIN']
  Component: React.FC;                 // the actual React component
}
```

A central `adminWidgets` registry exports an array of these. A new `<AdminSection />` page reads the registry, filters by current user role (defensive — the route gate already restricts to ADMIN), and renders. Adding a new admin widget is three lines plus the React component.

What's new in P2.1 vs P1.8:

1. **Role gating shifts from runtime-switch to route-gate.** P1.8's `<DashboardRouter />` is a runtime *switch* — both ENGINEER and CUSTOMER can hit `/`, and the component picks what to render. For `/admin`, the route itself must reject non-admins; a runtime switch leaks the existence of `/admin` to non-admins. P2.1 introduces a reusable `<RequireRole role="ADMIN" />` outlet wrapper that lives alongside `<RequireAuth />` and `<RequireActiveAuth />` in `frontend/src/routes/`.
2. **Two parallel widget registries.** `dashboardWidgets` (P1.8) and `adminWidgets` (P2.1) coexist. The widget contract is shared (`@/features/dashboard/types`) but the registries are separate arrays — they answer different questions. A future refactor could parameterize the page shell over a registry, but day-one duplication is the right call: each registry has its own audience semantics, ordering convention, and add-widget README.

Anti-goal: **no admin-specific backend module yet.** P2.1 extends `UsersController` with a single new endpoint (`GET /users?status=PENDING_APPROVAL`); a dedicated `AdminController` / `AdminModule` is premature until at least two admin endpoints share concerns. P2.2 adds the notification surface in its own `NotificationsModule`; P2.4 introduces `CustomersController`. If by P2.4 we have admin-only endpoints scattered across three controllers, that's the moment to consolidate — not now.

## Affected areas

### Backend — new files

- `backend/src/users/dto/list-users-query.dto.ts` — query DTO for the new list endpoint. Single optional field: `status?: UserStatus`. Validated with class-validator `@IsOptional() @IsEnum(UserStatus)`. Future fields (e.g., `?role=`) extend this DTO without breaking existing callers.

### Backend — modified files

- `backend/src/users/users.controller.ts` — add `GET /users` (admin-gated, tenant-scoped). Returns an array of `SafeUser`. Filter param `?status=PENDING_APPROVAL` is the P2.1 driver; absent status means "all users in tenant" (admins will want this view eventually, and it costs nothing to support).
- `backend/src/users/users.service.ts` — add `listByTenant(tenantId: string, filters: { status?: UserStatus }): Promise<User[]>`. Uses the existing TypeORM repo with a `where` clause. Tenant-scoped at the service layer, not the controller, so cross-tenant access returns an empty list rather than a 403 (404-not-403 pattern — consistent with `CasesService`).
- `backend/src/users/users.controller.spec.ts` — new test file (currently no controller-level test exists; spec covers the new endpoint plus exercises the existing `PATCH /:id/status` admin guard).
- `backend/src/users/users.service.spec.ts` — extend with `listByTenant` coverage (tenant isolation; status filter; ordering).

### Frontend — new files

**Routes:**

- `frontend/src/routes/RequireRole.tsx` — outlet wrapper, takes `role: UserRole` prop. If `user.role === role`, renders `<Outlet />`. Otherwise redirects to `/` via `<Navigate to="/" replace />`. Defensive null check on user (shouldn't happen below `<RequireActiveAuth />`, but documented).

**Admin feature folder:**

- `frontend/src/features/admin/types.ts` — re-exports `Widget` and `WidgetRegistry` from `@/features/dashboard/types`. Single source of truth for the contract; no new types in P2.1.
- `frontend/src/features/admin/registry.ts` — `adminWidgets: ReadonlyArray<Widget>`. Imports `PendingApprovalUsers` and exports the array. Mirrors `frontend/src/features/dashboard/registry.ts` shape.
- `frontend/src/features/admin/index.ts` — public barrel exporting `adminWidgets`.
- `frontend/src/features/admin/README.md` — "how to add an admin widget" adopter docs, mirroring `frontend/src/features/dashboard/README.md`. Architectural rules + naming conventions. Calls out the *one* difference from dashboard widgets: `audience` is always `['ADMIN']` (other roles get redirected at the route, so widgets don't need to defend).

**Users feature folder (shared cache for admin widgets that need user data):**

- `frontend/src/features/users/usersApi.ts` — axios wrappers. `list(filters?: { status?: UserStatus }): Promise<SafeUser[]>` calls `GET /users` with the query param.
- `frontend/src/features/users/usersSlice.ts` — Redux slice. `fetchUsersThunk(filters)` lazy-loads users on first request, keyed by serialized filter. Selectors: `selectPendingApprovalUsers(state)`, `selectUserById(state, id)`. Approval/rejection actions go through a separate `updateUserStatusThunk` that dispatches `usersSlice` updates on success.
- `frontend/src/features/users/types.ts` — `SafeUser` wire shape (mirrors backend `SafeUser` minus `passwordHash`). Includes `id`, `email`, `name`, `role`, `status`, `tenantId`, `createdAt`, `updatedAt`.
- `frontend/src/features/users/index.ts` — public barrel.

**Widget:**

- `frontend/src/features/admin/widgets/PendingApprovalUsers.tsx` — reads from `usersSlice`, filters by `status === PENDING_APPROVAL`. Renders a `<List>` of `<ListItem>`s with name + email + "registered Nd ago" + two action buttons (Approve, Reject). Buttons dispatch `updateUserStatusThunk({ userId, status: 'ACTIVE' | 'DISABLED' })`. Optimistic update: row disappears from the list immediately; rolls back on failure with a snackbar. Empty state: "No pending registrations." Loading state: row skeleton.

**Admin page:**

- `frontend/src/pages/Admin/Admin.tsx` — the page shell. Renders a welcome header (`<Typography variant="h4">Admin</Typography>` — anchor for future admin tour at `data-tour-id="admin.welcome"`), then reads `adminWidgets`, filters by role (defensive — should always pass since the route is ADMIN-gated), renders into a 2-column responsive `<Grid>`. First widget stretches full-width on small screens, pairs at `md={6}` and above. Same layout convention as `Dashboard.tsx`.
- `frontend/src/pages/Admin/index.ts` — barrel.

### Frontend — modified files

- `frontend/src/App.tsx` — add the `/admin` route inside the `<RequireActiveAuth />` + `<Layout />` group, wrapped in the new `<RequireRole role="ADMIN" />` outlet. Single multi-line addition; no existing routes change.
- `frontend/src/components/Layout/Layout.tsx` — add an "Admin" nav button in the app bar, rendered conditionally when `user?.role === 'ADMIN'`. Button links to `/admin` via `react-router`'s `<Link>` (styled as MUI Button). Positioned between the role/status chip and the Sign out button.
- `frontend/src/store/index.ts` — register the `users` reducer alongside the existing `cases`, `auth`, `tours` reducers.

### Tests

- `backend/src/users/users.controller.spec.ts` — new tests:
  - `GET /users` rejects non-admin callers with 403.
  - `GET /users?status=PENDING_APPROVAL` returns only PENDING_APPROVAL users in the actor's tenant.
  - `GET /users` (no status param) returns all users in the actor's tenant.
  - Cross-tenant isolation: Tenant B's pending users are invisible to Tenant A's admin.
  - `PATCH /users/:id/status` rejects PENDING_APPROVAL as the target status (existing DTO validation; one test confirms the rejection wires through).
- `backend/src/users/users.service.spec.ts` — extend:
  - `listByTenant` returns rows scoped to the given tenant only.
  - `listByTenant` with status filter returns only matching rows.
  - `listByTenant` with no filter returns all rows.
- `frontend/src/features/admin/widgets/PendingApprovalUsers.test.tsx`:
  - Renders empty state when no pending users.
  - Renders user rows when pending users exist.
  - Approve button dispatches `updateUserStatusThunk({ status: 'ACTIVE' })`.
  - Reject button dispatches `updateUserStatusThunk({ status: 'DISABLED' })`.
  - Row disappears optimistically on dispatch; reappears on rejection.
- `frontend/src/features/users/usersSlice.test.ts`:
  - `fetchUsersThunk` lazy-loads on first call; second call no-ops if already loaded.
  - `updateUserStatusThunk` updates the slice on success; reverts on failure.
  - Selectors filter correctly.
- `frontend/src/routes/RequireRole.test.tsx`:
  - Renders `<Outlet />` when `user.role === role`.
  - Redirects to `/` when `user.role !== role`.
  - Defensive null-user case redirects to `/login` (or wherever upstream `<RequireActiveAuth />` would have caught it — verify against actual behavior in test).
- `frontend/src/pages/Admin/Admin.test.tsx`:
  - Renders all `adminWidgets` for ADMIN users.
  - (Route-level test, possibly in App.test.tsx if it exists, that `/admin` redirects ENGINEER users to `/`.)

## Acceptance criteria

- [ ] An `ADMIN` user signing in sees an "Admin" link in the app bar. Clicking it navigates to `/admin`. The admin page renders the welcome header and the `PendingApprovalUsers` widget.
- [ ] An `ENGINEER` user signing in does **not** see the "Admin" link. Manually navigating to `/admin` redirects them to `/`.
- [ ] A `CUSTOMER` user signing in does **not** see the "Admin" link. Manually navigating to `/admin` redirects them to `/` (which routes via `<DashboardRouter />` to the `<Home />` placeholder).
- [ ] An unauthenticated user navigating to `/admin` hits `<RequireActiveAuth />` upstream and gets redirected to `/login`.
- [ ] `PendingApprovalUsers` widget shows only users with `status === PENDING_APPROVAL` in the actor's tenant, ordered by `createdAt DESC` (newest first).
- [ ] Approve button flips the user to `ACTIVE`. Reject button flips them to `DISABLED`. Both update via `PATCH /users/:id/status`. The row disappears from the list optimistically.
- [ ] Cross-tenant isolation: Tenant A's admin cannot see or act on Tenant B's pending users. Backend tests verify this at the controller layer.
- [ ] `GET /users` without a status param returns all users in the actor's tenant. Returned objects are `SafeUser` (no `passwordHash`).
- [ ] `GET /users` rejects non-admin callers with 403, consistent with the existing `PATCH /users/:id/status` admin guard.
- [ ] The widget-registry composition is reused, not re-implemented. `adminWidgets` lives in `frontend/src/features/admin/registry.ts` and shares the `Widget` contract from `@/features/dashboard/types`.
- [ ] `yarn workspace @caseflow-ai/frontend typecheck` clean.
- [ ] `yarn workspace @caseflow-ai/backend test` clean (existing + new tests).
- [ ] No proxy / CloudFront behavior changes needed (P2.1 consumes the existing `/users` API path, which is already proxied via the single-origin pattern from P10).

## Resolved pre-flight decisions

**Pre-flight 2 (P2) — Admin section composition reuses the widget-registry primitive.** Should be drafted into `docs/decisions/phase-02-admin-and-personalization.md` alongside this spec and committed in the same change. The decision captures:

- *Decision.* Reuse the `Widget` contract from `@/features/dashboard/types`. Two parallel registries (`dashboardWidgets`, `adminWidgets`) sharing the contract. New `<AdminSection />` page mirrors `<Dashboard />`'s structure (welcome header + grid of widgets) instead of inventing a new shell.
- *Alternative rejected — generic `<WidgetPage registry={...} />` shell.* Tempting because both pages do the same thing structurally, but the two pages have meaningfully different headers (Admin's header will grow to carry tenant-wide stats; Engineer's stays minimal) and different `data-tour-id` namespaces. Premature deduplication. Revisit when a third surface (e.g., a CUSTOMER-facing dashboard in P14) wants the same shape.
- *Alternative rejected — single combined `allWidgets` array with role-only filtering.* Would let `ENGINEER` and `ADMIN` both be audiences on the dashboard widgets, and `['ADMIN']` widgets would land on `/admin`. Rejected because role isn't the only axis — engineer dashboard and admin section answer different *questions*, not just have different audiences. A combined array invites accidental cross-audience widgets and obscures the architectural intent.

**Pre-flight 3 (P2) — Admin routing as a single `/admin` page (not `/admin/*` sub-routes) for now.** Should be drafted in the same change. Captures:

- *Decision.* `/admin` is a single route until widget count crosses ~5 and the page becomes cognitively crowded. At that point, split into `/admin/users`, `/admin/customers`, etc. with shared admin nav.
- *Alternative rejected — `/admin/*` sub-routes from day one.* Premature structure. With one widget, sub-routes are pure overhead.
- *Revisit if.* Admin widget count crosses 5, OR a single widget's complexity grows enough that it needs a dedicated URL (e.g., user detail drill-down). The widget-registry pattern doesn't preclude this — the future split is purely additive.

**Pre-flight 4 (P2) — Role gating as route wrapper (`<RequireRole>`) rather than runtime switch.** Should be drafted in the same change. Captures:

- *Decision.* New `<RequireRole role="ADMIN" />` outlet wrapper in `frontend/src/routes/`. Lives alongside `<RequireActiveAuth />` and `<RequireAuth />`. Used inside `App.tsx`'s route map to gate `/admin`.
- *Alternative rejected — `<AdminRouter />` runtime switch (mirroring `<DashboardRouter />`).* P1.8's `<DashboardRouter />` is a *switch* (both ENGINEER and CUSTOMER can hit `/`; the component picks what to render). For `/admin`, the route itself must reject non-admins — a runtime switch would leak the URL's existence and force every non-admin pageload through the admin code path. Different semantic: switch for shared URL, gate for restricted URL.
- *Revisit if.* The pattern proves awkward when role-gated routes need to render different fallbacks (some redirect to `/`, some to a 404, some to a "request access" page). At that point, the wrapper grows a `fallback` prop or splits into multiple variants.

## Out of scope (deferred)

- **Admin notification on user registration.** Lands in P2.2. P2.1 ships the surface where the email's "view pending users" deep-link will land.
- **Customer organizations / customer model.** Lands in P2.3+. P2.1 deliberately holds the admin surface to a single widget so the widget-registry payoff is visible when P2.4 adds the second widget.
- **User detail drill-down page.** Would live at `/admin/users/:id`. Deferred until the list view's information density is insufficient for the operator's needs. The list widget's row UI is intentionally information-dense to push this deferral.
- **Bulk approve / reject.** Single-row actions only. Bulk operations add a "select multiple" UI affordance + a confirmation modal that don't earn their complexity until the pending volume justifies it. Revisit if the live demo's pending queue regularly exceeds ~10 users.
- **Audit trail of approval actions.** Who approved whom, when. Belongs in a future `audit_log` table — premature in P2.1.
- **Admin onboarding tour.** The existing `onboarding.engineer` tour (v2 from P1.8) does not need to extend. A separate `onboarding.admin` tour could land later, but ADMINs in this build are also engineers (they see both `/` and `/admin`) — the engineer tour already orients them. Add a one-step "Find the Admin section in the top bar" prompt to the engineer tour for ADMIN users only if user testing shows admins miss the nav link.
- **Filtering / search within the pending list.** With volumes <50 the list is fine as-is. Filtering UI lands when volumes (or operator request) justify it.

## How we'll work this sub-phase

Three chunks with review checkpoints between each. Same pattern as P1.7 and P1.8.

### Chunk A — Backend list endpoint + tests

1. `users.service.ts` — add `listByTenant` method.
2. `users.controller.ts` — add `GET /users` endpoint, admin-gated, tenant-scoped, with optional `?status=` query.
3. `users/dto/list-users-query.dto.ts` — validation DTO.
4. `users.controller.spec.ts` — new file covering the new endpoint + the existing PATCH guard.
5. `users.service.spec.ts` — extend with `listByTenant` coverage.
6. Verify via curl / `dev.sh db:psql` that the endpoint returns the expected shape.

**Review checkpoint 1.** Endpoint contract reviewed. Tenant isolation verified by integration test. DTO matches the agreed shape.

### Chunk B — Frontend route gate + admin page shell + widget

1. `frontend/src/routes/RequireRole.tsx` — outlet wrapper with tests.
2. `frontend/src/features/users/` — `usersApi.ts`, `usersSlice.ts`, `types.ts`, `index.ts`, tests.
3. `frontend/src/features/admin/` — `types.ts` (re-export), `registry.ts` (empty array initially), `index.ts`, `README.md`.
4. `frontend/src/pages/Admin/` — `Admin.tsx` page shell.
5. `App.tsx` — wire the `/admin` route under `<RequireRole role="ADMIN" />`.
6. `Layout.tsx` — add the "Admin" nav button, role-conditional.
7. `frontend/src/features/admin/widgets/PendingApprovalUsers.tsx` — implement the widget.
8. `registry.ts` — register the widget.
9. Tests for slice, widget, route wrapper, admin page.

**Review checkpoint 2.** Admin link visible only to ADMIN users. `/admin` renders the page with one widget. Approve and reject buttons round-trip to the backend and update the UI optimistically. Network tab shows exactly one `GET /users?status=PENDING_APPROVAL` request per admin page load (the slice's lazy-load).

### Chunk C — Documentation + decision-log pre-flights

1. Draft Pre-flight 2, 3, 4 in `docs/decisions/phase-02-admin-and-personalization.md` (the bodies are stubbed above in "Resolved pre-flight decisions").
2. Update the dashboard `README.md` to note the existence of the admin section + cross-link to `frontend/src/features/admin/README.md`.
3. ROADMAP — flip P2.1 from `⬜` to `✅` and add the implementation date.

**Review checkpoint 3.** Pre-flights 2-4 captured in the phase-02 log. Cross-link from dashboard README to admin README. ROADMAP updated.

## Risks and how we'll handle them

- **Optimistic-update rollback ergonomics.** Approving a user removes the row instantly. If the `PATCH` fails (network, race with another admin acting on the same row, etc.), the slice rolls back the local state and surfaces a snackbar. The rollback path is the easy-to-miss case in widget testing — explicit test coverage required. Snackbar uses MUI's `<Snackbar>` with a 4-second auto-hide; copy says "Couldn't update [name]'s status — try again." Avoid blaming the user.
- **Two parallel registries, divergent over time.** `dashboardWidgets` and `adminWidgets` share a contract today; if either grows fields the other doesn't need, the contract drifts. Mitigation: keep the contract minimal (the existing `{ id, title, audience, Component }`), document the no-divergence rule in both READMEs, and revisit if a third registry surface ever wants a third subtly-different shape.
- **`/admin` indexed by search engines or shared accidentally.** The route is auth-gated, but the URL itself is meaningful. Mitigation: no robots.txt change required (the page never renders without auth), but worth flagging in a future P12 prod-hardening review that admin URLs should add `<meta name="robots" content="noindex">` defensively.
- **Approve / reject button mis-tap.** Two adjacent buttons doing opposite things invite a mis-tap. Reject is destructive (sets DISABLED). Mitigation v1: Reject opens a confirmation modal ("Reject [name]? They'll be unable to log in. This is reversible from the user detail page later.") — the small friction is worth it. Approve goes direct (low-risk; reversible by flipping to DISABLED). The modal lives in the widget for now; if other admin widgets need a similar confirm-destructive pattern, extract.
- **Pending user list grows past one screen.** P2.1 caps the visual at no pagination; the list renders all returned rows. With volumes <30, that's fine. If the live demo's pending queue regularly exceeds 30, P2.5-ish work adds pagination — but that's a "wait and see" call, not a P2.1 concern.
- **Future expansion of `GET /users`.** The endpoint returns all users in the tenant by default. As tenants grow, that response gets large. Mitigation in P2.1: cap the response at 200 users (TypeORM `take: 200`) with an ordered-by-`createdAt-DESC` to ensure the most recent (and most relevant) are returned. Above the cap, a hard limit response leaves room for the user to add pagination in a future iteration. The cap is a defensive measure; the test suite confirms the limit applies.

## Followup

After P2.1 ships:

- P2.2 (Admin notification on user registration) is the natural next step. With the admin surface in place, the notification email's deep-link target exists.
- P2.4 will register `CustomerOrganizations` as the second admin widget — the moment the widget-registry payoff becomes visible on the admin side. That's a natural article anchor for the series (companion content opportunity).
- The new `<RequireRole>` wrapper unblocks any future role-gated routes (engineer-only admin tools, customer-only surfaces in P14, etc.).
- ROADMAP P2 status flips to `█░░░░░░░░░ 14% 1/7` after P2.1.
- Content series: this spec is a candidate second-example reference (alongside P1.7 and P1.8) for **Article 11 — Dev-ready specs for AI pair programming**, particularly the "two parallel registries, shared contract" pattern.
