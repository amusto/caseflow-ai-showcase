# Dev-Ready Task — P1.7 · Internal-only Tour infrastructure

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.7**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md#pre-flight-23`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Ship a tour engine that walks **internal users** (ENGINEER + ADMIN, never CUSTOMER) through the app post-login. The infrastructure must support three trigger styles — auto on first login, manual via "Take the tour" button, event-driven when a tour version bumps — and must make adding a tour for a new feature a single TypeScript file dropped under `frontend/src/tours/`.

P1.7 ships the engine, the targeting convention, the state persistence layer, and one real tour (the onboarding walkthrough of the post-login surface). It does NOT ship feature-specific tours for P2+ work — those land with their features.

## Why this architecture (the 90-second mental model)

The system has three moving parts, each isolated from the other two:

1. **Targeting.** Every UI element a tour might point at carries a `data-tour-id="domain.element"` attribute. This is the DOM contract. CSS classes and component names change; `data-tour-id` is a deliberate API.
2. **Definitions.** Each tour is a TypeScript object in `frontend/src/tours/`, declaring its id, version, audience, trigger, and step list. Definitions live in source — they ship in the same PR as the feature they describe, get code-reviewed, version in git, and never go out of sync with the UI.
3. **Engine.** A single `<TourEngine />` mounted inside `<Layout />` reads tour state from Redux, decides which tours to offer the current user, and renders `react-joyride`'s overlay. The engine is the only component that talks to react-joyride. Every tour gets the same renderer.

**Internal-only is a render-time decision, not a route-time decision.** The engine itself checks `user.role === ENGINEER || ADMIN` before rendering anything. Why: the engine sits inside `<Layout />` which is already behind `RequireActiveAuth`, so the route guard is satisfied, but `<Layout />` will eventually host customer-role surfaces (P14). Putting the gate inside the engine keeps the rule close to the rendering decision.

State persistence is server-side because it's a single table and two endpoints — and because localStorage doesn't sync across devices, which would mean a user who completed onboarding on their laptop sees it again on their phone. The senior call.

## Affected areas

### Backend — new files

- `backend/src/tours/entities/user-tour-state.entity.ts` — `UserTourState` entity with composite-unique `(userId, tourId)`:
  - `id: uuid` (PK)
  - `userId: uuid` (FK → users.id, indexed, NOT NULL)
  - `tenantId: uuid` (FK → tenants.id, indexed, NOT NULL — needed for tenant-scoped queries + future analytics)
  - `tourId: string` (e.g. `"onboarding.engineer"`, NOT NULL)
  - `version: integer` (the tour version the user saw; NOT NULL)
  - `status: enum('started' | 'completed' | 'dismissed')` (NOT NULL)
  - `completedAt: timestamp | null`
  - `createdAt`, `updatedAt` (automatic)
  - Unique index on `(userId, tourId)` — one row per user-tour pair; updating a tour's version updates this row in place.
- `backend/src/tours/dto/upsert-tour-state.dto.ts` — body for `POST /tour-state`. Fields: `tourId` (string, length-bounded), `version` (integer ≥ 1), `status` (enum). Validated via class-validator.
- `backend/src/tours/dto/tour-state.dto.ts` — response shape; subset of entity fields safe to return.
- `backend/src/tours/tours.service.ts` — `findAllForUser(userId, tenantId)` and `upsert({ userId, tenantId, tourId, version, status })`. Tenant-scoped per the project-wide invariant.
- `backend/src/tours/tours.controller.ts` — `@Controller('tour-state')` with `GET /` (returns the current user's full tour-state record set) and `POST /` (upsert one tour's state). Both protected by the global JwtAuthGuard + ActiveUserGuard — no `@Public()` decorator. **No role gate at the controller layer** — `CUSTOMER` users can technically call these endpoints, they just have no tours to read or write. Putting the role gate here would create a false sense of security; the real gate is on the rendering side.
- `backend/src/tours/tours.module.ts` — module declaration + TypeORM repository registration.

### Backend — modified files

- `backend/src/app.module.ts` — register `ToursModule`.

### Frontend — new files

- `frontend/src/features/tours/types.ts` — TypeScript types:
  ```ts
  type TourStep = {
    target: string;            // `data-tour-id` value, e.g. "cases.create-button"
    title: string;
    body: string;              // markdown allowed
    placement?: 'top' | 'bottom' | 'left' | 'right' | 'auto';
    spotlightClicks?: boolean; // can the user click the highlighted element?
  };

  type TourTrigger =
    | { kind: 'first-login' }
    | { kind: 'manual' }
    | { kind: 'event'; name: string };

  type TourDefinition = {
    id: string;                // dot-namespaced, e.g. "onboarding.engineer"
    version: number;           // bump to re-show after content changes
    audience: ('ENGINEER' | 'ADMIN')[];   // never includes 'CUSTOMER'
    trigger: TourTrigger;
    steps: TourStep[];
  };
  ```
- `frontend/src/features/tours/registry.ts` — central registry: imports each tour definition file and exports `allTours: TourDefinition[]`. Adding a new tour means adding an import here. Lint-able list.
- `frontend/src/features/tours/toursSlice.ts` — Redux slice:
  - State: `{ records: UserTourStateRecord[], loading, error, activeTourId: string | null }`.
  - Thunks: `fetchTourStateThunk` (calls `GET /tour-state`), `upsertTourStateThunk` (calls `POST /tour-state`).
  - Actions: `startTour(tourId)`, `dismissActive()`, `completeActive()`.
- `frontend/src/features/tours/toursApi.ts` — axios calls to `/tour-state` GET + POST.
- `frontend/src/features/tours/TourEngine.tsx` — the engine. Responsibilities:
  1. On mount (after auth bootstrap), dispatch `fetchTourStateThunk`.
  2. Compute "tours to offer right now" given current user role + route + records:
     - Auto-trigger: `first-login` tours whose id is not in `records` and audience includes the user's role.
     - Event-trigger: tours whose record's `version < definition.version`.
     - Manual-trigger: shown only on `startTour(tourId)` dispatch (the "Take the tour" button).
  3. Render `<Joyride ...>` for the active tour. The Joyride callback dispatches `upsertTourStateThunk` on completion or dismissal.
  4. **Role gate** at the top of render: if `user.role === 'CUSTOMER'`, return `null` and skip the entire fetch. Never renders, never asks the backend for state.
- `frontend/src/features/tours/tours/onboarding-engineer.ts` — first real tour. 4 steps anchored to the placeholder Home page surface (user info card, the future "create case" zone — we ship the `data-tour-id` attrs even though the dashboard widget hasn't landed yet so P1.8 doesn't have to re-author the tour). Anti-goal: do not anchor steps to the dashboard widgets — those don't exist yet in P1.7's branch.
- `frontend/src/features/tours/README.md` — adopter docs. How to add a new tour, the `data-tour-id` convention, and how the engine decides what to show. **The user reads this.** It must be clear.
- `frontend/src/features/tours/index.ts` — barrel export.

### Frontend — modified files

- `frontend/src/components/Layout.tsx` — mount `<TourEngine />` inside the layout (after the routed children render so the targets exist in the DOM).
- `frontend/src/store/index.ts` — add `tours` reducer.
- `frontend/src/pages/Home/Home.tsx` — add `data-tour-id` attributes to the existing placeholder elements so the onboarding tour has targets to point at. Five attributes: `home.welcome-heading`, `home.user-info`, `home.tenant-info`, `home.role-info`, `home.status-info`. **Anti-goal**: do NOT redesign Home in this sub-phase. P1.8 replaces it with the dashboard. P1.7 just adds attributes.
- `frontend/src/pages/AwaitingApproval/AwaitingApprovalPage.tsx` — single `data-tour-id="awaiting-approval.banner"` on the main message. Lets a future "you're awaiting approval, here's what happens next" tour anchor here.
- `frontend/package.json` — add `react-joyride` to dependencies.

### Tests

**Backend:**
- `tours.service.spec.ts` — `findAllForUser` returns only the caller's records; `upsert` creates on first call, updates on second.
- `tours.controller.spec.ts` — `GET /tour-state` happy path; `POST /tour-state` validation (rejects bad `status` enum value, missing fields).
- Tenant scoping assertion: a record's `tenantId` always matches the caller's `tenantId`; a record from Tenant B is invisible to a user in Tenant A even with the same user ID by some forgery.

**Frontend:**
- `TourEngine.test.tsx` — role gate test: when `user.role === 'CUSTOMER'`, engine renders nothing AND does not dispatch `fetchTourStateThunk`.
- `TourEngine.test.tsx` — first-login auto-trigger: with empty records, the matching first-login tour activates.
- `TourEngine.test.tsx` — version bump: with a stored record at version 1 and a definition at version 2, the tour re-activates.
- `toursSlice.test.ts` — thunk happy paths.

## Acceptance criteria

- [ ] `UserTourState` migration runs cleanly on a fresh DB (or `synchronize: true` recreates the table from the entity).
- [ ] `GET /tour-state` returns `[]` for a user who has never completed any tour.
- [ ] `POST /tour-state` upserts: first call creates, second call with same `tourId` updates `status` + `version` in place.
- [ ] An `ENGINEER` or `ADMIN` user logging in for the first time sees the onboarding tour overlay automatically. Walking through to completion records `status='completed'` in the backend.
- [ ] A `CUSTOMER` user logging in sees **no tour overlay** and the engine never calls `GET /tour-state` (assert via network mock).
- [ ] Refreshing the page after completing a tour does NOT re-show it.
- [ ] Bumping the version of the onboarding tour in source and redeploying re-shows the tour to users whose record's version is lower (manually verifiable; tested in unit tests).
- [ ] `frontend/src/features/tours/README.md` documents the `data-tour-id` naming convention, the tour definition shape, and the "how to add a tour" workflow in under 200 words. User can read it and add a stub tour without asking questions.
- [ ] `yarn workspace @caseflow-ai/backend lint && typecheck && test` clean.
- [ ] `yarn workspace @caseflow-ai/frontend lint && typecheck && test` clean.

## Resolved pre-flight decisions (see decision log)

- **Pre-flight 23 — Tour engine architecture.** react-joyride + `data-tour-id` + source-controlled tour definitions + server-side `UserTourState`. Internal-only audience gate enforced in the engine, not the route guard.

## Out of scope (deferred)

- Feature-specific tours for P2+ surfaces — those ship with their features.
- i18n / multi-language tour content — string literals only in v1; types shaped to accommodate translation keys later.
- Per-tenant tour customization — defer until a real product requirement lands.
- Tour analytics / completion funnels — defer to P11 observability.
- Customer-facing onboarding — out of scope by design (Pre-flight 23). When P14 customer app lands, it gets its own engine.

## How we'll work this sub-phase

Three chunks with review checkpoints between each. The user breaks for review at every checkpoint; if you walk away mid-implementation, the next chunk picks up cleanly without re-reading the prior code.

### Chunk A — Backend tour-state surface
1. `UserTourState` entity + DTOs + service + controller + module + tests.
2. Wire `ToursModule` into `AppModule`.
3. `synchronize: true` picks up the new table on backend restart.

**Review checkpoint 1.** Show the user:
- Entity definition.
- One paragraph in plain English explaining the tenant-scoping rule for this entity and why the controller has no role gate.
- Curl-able example: log in, `curl GET /tour-state` returns `[]`, `curl POST /tour-state` with a stub body returns the created record, `curl GET` returns the record.

### Chunk B — Frontend tour engine
1. Add `react-joyride` dependency, run `yarn install`.
2. Build `features/tours/` directory: types, registry, slice, API client, engine, README.
3. Mount engine in Layout.
4. Wire Redux store.

**Review checkpoint 2.** Show the user:
- The engine component, walked through line by line.
- The README — confirm it's clear enough to onboard a teammate.
- A unit test demonstrating the role gate works.

### Chunk C — First onboarding tour
1. Add `data-tour-id` attributes to Home + AwaitingApproval pages.
2. Author `onboarding-engineer.ts` tour definition (4 steps walking the placeholder Home surface).
3. Register it in `registry.ts`.

**Review checkpoint 3.** User logs in as a fresh ENGINEER on local dev, sees the tour fire, walks through it. Then logs in as a CUSTOMER (the seeded customer from the existing default tenant or a fresh registration) and confirms no overlay appears.

After checkpoint 3 passes, commit + push as `feat(P1.7): internal-only tour infrastructure + onboarding tour`.

## Risks and how we'll handle them

- **react-joyride styling clashes with MUI's z-index stack.** Mitigation: set Joyride's `styles.options.zIndex` higher than MUI's modal default (`1300`). Use `10000`. If the dropdown menu inside MUI's app bar starts appearing above the tour overlay, that's the symptom.
- **Tour targets don't exist at engine mount time.** If the user is on `/awaiting-approval` but the active tour anchors to `home.welcome-heading`, react-joyride either errors or silently skips. Mitigation: tour definitions get an optional `route?: string` field in the future; for P1.7 we only ship a tour whose targets are on the route where the engine fires.
- **`data-tour-id` collisions.** Mitigation: namespace by domain (`cases.X`, `home.X`). Add a linter rule in a future sub-phase if drift surfaces.

## Followup

After P1.7 ships:
- P1.8 dashboard MVP rewrites the Home page. The onboarding tour gets re-anchored to dashboard widgets in the same PR — the engine and the tour definition file stay; only the `target` values change. Cheap pivot precisely because of the engine/definition split.
- Content series Article 11 ("Dev-ready specs for AI pair programming") gets this spec as a concrete example.
