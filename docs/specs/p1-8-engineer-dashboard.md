# Dev-Ready Task — P1.8 · Engineer Dashboard MVP

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.8**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md#pre-flight-24`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Replace the placeholder `<Home />` page with a **widget-composed Engineer Dashboard** at `/` for `ENGINEER` and `ADMIN` users. The dashboard becomes the post-login landing surface — what every demo viewer, recruiter, and stakeholder sees first when they hit the deployed environment.

CUSTOMER users continue to land on the existing `<Home />` placeholder for now. P14 formalizes the customer-facing app.

P1.8 ships three MVP widgets (`myOpenCases`, `recentActivity`, `quickActions`) using only data sources that already exist from P1.1–P1.5. Each subsequent phase adds widgets to the registry without touching the page shell — that's the architectural payoff.

## Why this architecture (the 90-second mental model)

The whole dashboard hinges on **one boring contract** — the `Widget` interface:

```ts
interface Widget {
  id: string;                          // 'myOpenCases'
  title: string;                       // 'My open cases'
  audience: UserRole[];                // ['ENGINEER', 'ADMIN']
  Component: React.FC;                 // the actual React component
}
```

A central registry exports an array of these. The dashboard page reads the registry, filters by current user role, renders each widget. Adding a new widget is three lines of code plus the React component.

Crucially: **widgets fetch their own data.** No prop-drilling, no parent-aware contract. Two widgets that need the same data (e.g., `myOpenCases` and `recentActivity` both need `/cases`) share a Redux slice that lazy-loads once. That's the cache pattern Pre-flight 24 called for — explicitly chosen over a server-rendered "give me everything" endpoint.

The pattern is a near-copy of the tour engine's registry pattern (P1.7) — same shape, same rationale, same payoff for future phases.

## Affected areas

### Frontend — new files

**Dashboard feature folder:**

- `frontend/src/features/dashboard/types.ts` — TypeScript contracts: `Widget`, `WidgetRegistry`. Audience filtered to `UserRole` from `@/features/auth/types`.
- `frontend/src/features/dashboard/registry.ts` — central list, imports each widget component and exports `allWidgets: Widget[]`. Mirrors the tours registry.
- `frontend/src/features/dashboard/index.ts` — public barrel.
- `frontend/src/features/dashboard/README.md` — adopter docs for "how to add a widget" workflow. Architectural rules + naming conventions.

**Cases feature folder (shared cache for widgets that need cases):**

- `frontend/src/features/cases/casesApi.ts` — axios wrappers. `list()` calls `GET /cases`.
- `frontend/src/features/cases/casesSlice.ts` — Redux slice. `fetchCasesThunk` lazy-loads cases on first request; second call no-ops if already loaded (returns cached). Selectors: `selectMyOpenCases(state, userId)`, `selectRecentlyUpdatedCases(state, limit)`.
- `frontend/src/features/cases/types.ts` — `Case` wire shape mirroring the backend entity (id, title, description, status, priority, assignedTo, tenantId, createdAt, updatedAt).
- `frontend/src/features/cases/index.ts` — public barrel.

**Widgets:**

- `frontend/src/features/dashboard/widgets/MyOpenCases.tsx` — reads from cases slice, filters by `assignedTo === user.id` AND `status !== RESOLVED`. Renders a `<List>` of `<ListItem>`s with title, status chip, and "age (e.g. '2d ago')". Empty state: "No cases assigned. Browse all cases →" (link stubbed for P2+).
- `frontend/src/features/dashboard/widgets/RecentActivity.tsx` — reads from cases slice, returns the 5 most recently updated cases regardless of assignment. Each row shows title + "Updated 2h ago" + status chip. Empty state: "No recent activity yet."
- `frontend/src/features/dashboard/widgets/QuickActions.tsx` — two controls. "Create case" button opens an MUI Dialog placeholder ("Full case-creation form lands in P2 with the customer model. Until then, use `POST /cases` from the API."). Search box is a disabled `<TextField placeholder="Search (P5+)">`.

**Dashboard page:**

- `frontend/src/pages/Dashboard/Dashboard.tsx` — the page shell. Renders the role-appropriate widgets from the registry in a 2-column responsive `<Grid>`. MUI `<Grid container spacing={3}>` with each widget wrapped in `<Grid item xs={12} md={6}>`. QuickActions stretches full-width (`xs={12} md={12}`) at the top; MyOpenCases + RecentActivity sit side-by-side below.
- `frontend/src/pages/Dashboard/index.ts` — barrel.

**Dashboard router (role switch):**

- `frontend/src/pages/Home/DashboardRouter.tsx` — single component at the `/` route. Reads `auth.user.role`; renders `<Dashboard />` for ENGINEER/ADMIN, `<Home />` (the existing placeholder) for CUSTOMER. Loading state if user is null (shouldn't happen given RequireActiveAuth, but defensive).

### Frontend — modified files

- `frontend/src/App.tsx` — route at `index` swaps from `<Home />` to `<DashboardRouter />`. Single-line change.
- `frontend/src/pages/Home/index.ts` — exports both `<Home />` (unchanged) and `<DashboardRouter />` so external imports don't have to know the file layout.
- `frontend/src/store/index.ts` — register the `cases` reducer.
- `frontend/src/tours/onboarding-engineer.ts` — **bump `version: 1 → 2`** and re-anchor the 4 steps to dashboard widgets. New targets: `dashboard.welcome`, `dashboard.my-open-cases`, `dashboard.recent-activity`, `dashboard.quick-actions`. Step copy refreshed to match the new surface. The version bump triggers a re-show for users who already completed v1.

### Tests

- `frontend/src/features/dashboard/registry.test.ts` — registry filters by audience correctly (CUSTOMER sees no widgets even if a widget mistakenly listed them).
- `frontend/src/features/cases/casesSlice.test.ts` — `selectMyOpenCases` filters correctly; `selectRecentlyUpdatedCases` returns N most recent ordered desc.
- `frontend/src/pages/Dashboard/Dashboard.test.tsx` — renders the right widgets per role (ENGINEER sees all 3, CUSTOMER would see 0 if it ever rendered — though DashboardRouter prevents that).
- `frontend/src/features/dashboard/widgets/MyOpenCases.test.tsx` — assignedTo + status filtering; empty state renders correctly.

## Acceptance criteria

- [ ] An `ENGINEER` or `ADMIN` user signing in lands on the new Engineer Dashboard at `/`. Three widgets visible: QuickActions on top, MyOpenCases + RecentActivity side-by-side below.
- [ ] A `CUSTOMER` user signing in continues to see the existing `<Home />` placeholder (P14 will eventually replace this).
- [ ] `myOpenCases` widget shows only cases where `assignedTo === user.id` AND `status !== RESOLVED`, sorted by priority then age.
- [ ] `recentActivity` widget shows 5 most-recently-updated cases for the tenant, regardless of assignment.
- [ ] `quickActions` widget's "Create case" button opens a placeholder dialog explaining the P2 dependency. Search input is disabled with placeholder copy.
- [ ] Both data-driven widgets (`myOpenCases`, `recentActivity`) share a single `casesSlice` cache — opening the dashboard fires `GET /cases` **exactly once**, not once per widget. Verified by network mock in tests.
- [ ] Empty states render cleanly when the user has no cases assigned and/or the tenant has no cases.
- [ ] **Tour re-anchored** — the `onboarding.engineer` tour at version 2 walks the user through `dashboard.welcome`, `dashboard.my-open-cases`, `dashboard.recent-activity`, `dashboard.quick-actions`. Users who completed v1 see v2 on next login. Users seeing for the first time see v2 directly.
- [ ] `yarn workspace @caseflow-ai/frontend typecheck` clean.
- [ ] No new backend endpoints introduced (per Pre-flight 24 anti-goal).
- [ ] No proxy / CloudFront behavior changes needed (P1.8 consumes existing `/cases` route, already proxied).

## Resolved pre-flight decisions

- **Pre-flight 24 — Engineer Dashboard composition.** Widget-registry pattern. MVP three widgets consuming existing data. Each subsequent phase adds widgets without touching the shell. Anti-goal: no new backend endpoints in P1.8.

## Out of scope (deferred)

- **Server-rendered composite dashboard endpoint.** Premature optimization. Each widget hits its own endpoint with frontend-side caching. Revisit only when dashboard load time becomes a measured concern.
- **Widget reordering / personalization.** A future feature where users could drag widgets to reorder, hide some, etc. The registry pattern doesn't preclude this; we just don't ship it in MVP.
- **Real search functionality.** Stubbed until P5+ when the entity surface justifies it.
- **Real Create Case form.** Deferred to P2 with the customer model — case creation needs a customer org picker, which doesn't exist until P2 ships.
- **Customer Dashboard.** `CUSTOMER` users stay on the placeholder `<Home />`. P14 formalizes the customer app.

## How we'll work this sub-phase

Three chunks with review checkpoints between each. Same pattern as P1.7.

### Chunk A — Page shell + widget registry + QuickActions
1. Dashboard feature folder: types, registry (empty), index, README.
2. `Dashboard.tsx` page that reads the registry and renders.
3. `DashboardRouter.tsx` that switches on role.
4. `QuickActions` widget — simplest, no data fetching.
5. App.tsx route swap.
6. Verify routing works: ENGINEER sees empty dashboard + QuickActions widget; CUSTOMER sees old Home.

**Review checkpoint 1.** Empty dashboard renders. QuickActions widget visible. Role-based routing confirmed.

### Chunk B — Cases slice + data-driven widgets
1. `frontend/src/features/cases/` — types, api, slice with lazy-load thunk and two selectors.
2. `MyOpenCases` widget — registers in the registry, dispatches `fetchCasesThunk` on mount, renders filtered list.
3. `RecentActivity` widget — same pattern, different selector.
4. Tests for slice + widgets.

**Review checkpoint 2.** Dashboard renders all three widgets with real case data from the dev environment. Network tab shows exactly one `GET /cases` request per dashboard load.

### Chunk C — Re-anchor onboarding tour to dashboard widgets
1. Add `data-tour-id` attributes to dashboard surfaces (`dashboard.welcome`, `dashboard.my-open-cases`, `dashboard.recent-activity`, `dashboard.quick-actions`).
2. Update `frontend/src/tours/onboarding-engineer.ts`: bump version to 2, rewrite step targets + copy.
3. Test: log in as a user who has already completed v1; the tour should re-fire because `version` 2 > stored 1. After completion, refresh — tour does NOT re-fire.

**Review checkpoint 3.** User drives the new tour against the new dashboard end-to-end.

## Risks and how we'll handle them

- **Widget render order.** MUI `<Grid>` renders in DOM order, so order depends on registry array order. Predictable. If a future widget needs explicit positioning, add an `order: number` field to the `Widget` contract — non-breaking addition.
- **Empty `/cases` response.** Widgets must handle `[]` cleanly. Empty states are required per acceptance criteria.
- **Tour anchoring race.** Tour engine evaluates triggers after `<Outlet />` renders. Dashboard widgets fetch on mount; if a tour step targets `data-tour-id="dashboard.my-open-cases"` and the widget is still loading, react-joyride might fail to find the target. **Mitigation:** the `data-tour-id` lives on the widget's outer `<Paper>` container which renders immediately regardless of data state. The widget's loading skeleton renders INSIDE the Paper, so the target always exists.
- **Two widgets, one request.** Both `MyOpenCases` and `RecentActivity` dispatch `fetchCasesThunk` on mount. Without dedup, that's two `/cases` calls. **Mitigation:** the thunk checks a `loading` flag inside its prefetch step; if loading or already-fetched, it returns early. Standard Redux Toolkit lazy-load pattern.

## Followup

After P1.8 ships:
- P1.9 (Demo seed harness) is the natural next step. With seed data populating the dashboard, the deployed env becomes a real demo surface.
- Content series Article 11 ("Dev-ready specs for AI pair programming") gets this spec as a second example alongside P1.7.
- ROADMAP P1 status flips to `████████░░░ 89% 8/9` after P1.8.
