# Phase 2 — Admin Foundation + Dashboard Personalization · Decision Log

**Phase:** [P2](../../ROADMAP.md#phase-2--admin-section--registration-notification-)
**Opened:** 2026-05-21 (decision-staging — no P2 code has landed yet)
**Status:** Open — decisions being captured ahead of the first sub-phase
spec.

This log captures the irreversible-shaped decisions that will frame
Phase 2. P2's working scope is the admin section (admin foundation,
registration-notification surface) plus dashboard personalization
(drag-and-drop layout, persisted per-user). Pre-flights here may be
written before the matching sub-phase opens — they exist to lock in
direction so the dev-ready specs stay short and the implementation
chunks stay mechanical.

---

## Pre-flight 1 — Dashboard drag-and-drop direction

**Date:** 2026-05-21 (forward-looking — drafted during P1 close-out, no
implementation work scheduled yet; expected sub-phase ~P2.5 after the
admin foundation lands).

**Decision.** Ship dashboard drag-and-drop in two layered steps, with
the smaller step first.

*Step 1 — reorder-only.* Users can drag widgets to change their order
on the page. Widgets keep the current sizing rules (first widget full
width on small screens, subsequent widgets pair up at `md`). Library:
[`@dnd-kit/core`](https://dndkit.com) + `@dnd-kit/sortable`. The
filtered widget array becomes a sortable list; each widget renders
inside a `<SortableWidgetCard>` that exposes a drag handle in the card
header so the widget body's interactive content stays clickable.

*Step 2 — grid + resize.* Only if users ask for it. At that point the
sortable wrapper is swapped for `react-grid-layout`, the contract
extends (`defaultLayout?: { w, h, minW, minH }`), and the persisted
shape grows from an `order: string[]` to a full layout array.

**Where state lives.** Two layers.

- *Hot state* — a new `dashboardLayoutSlice` in Redux. The slice
  applies user moves optimistically and debounces the persist call.
- *Cold state* — server-side, in a new `dashboard_layouts` table
  (`user_id`, `tenant_id`, `layout jsonb`, `updated_at`). Tenant-scoped
  the same way `cases` is; 404-not-403 if a user requests another
  user's row.

**API surface.** Two endpoints.

- `GET /dashboard/layout` — returns the saved layout for the
  authenticated user, or `null` if they haven't customized.
- `PATCH /dashboard/layout` — upserts. Body validated against the
  step-1 shape (`{ order: string[] }`) so unknown widget ids 400
  out cleanly.

**Reconciliation rule on the client.** The server's saved layout is
permission to position widgets — it is *not* the source of truth for
which widgets exist. On dashboard mount the slice takes the registry
(filtered by role, as today) and merges:

- Widgets the user has positioned → use their saved position
- Widgets added to the registry since they last saved → append at the
  bottom with defaults
- Widgets removed from the registry since they last saved → drop
  silently

That merge logic lives in the slice, not in `Dashboard.tsx`.

**Component shape.** `Dashboard.tsx` stays thin. The current
`<Grid container>` becomes a `<SortableDashboard widgets={visibleWidgets} layout={layout} onChange={persist} />`
component that owns the dnd-kit context, the drag handles, and the
optimistic-update wiring. The role filter and the registry read stay
exactly where they are today.

**Cost estimate.** Step 1 (reorder-only via dnd-kit, dedicated layouts
table): ~1.5 sub-phases worth of work. Backend (table migration + two
endpoints + tests) is half a day; frontend (slice + sortable wrapper +
Dashboard refactor + tests) is a day. Step 2 (grid + resize via
`react-grid-layout`) roughly doubles the frontend cost.

**Alternative rejected — `react-grid-layout` on day one.** Considered
shipping the full grid + resize experience from the first sub-phase.
Rejected because:

1. Resize is the smaller half of the user value — *what order are my
   widgets in* is the question users actually ask. Reorder solves it
   without the responsive-grid math, the drag-from-corner ergonomics,
   or the more complex persisted shape.
2. The library is heavier (its bundle, its CSS, its breakpoint
   awareness) and pre-empts choices we haven't validated — for
   example, "are widgets allowed to overlap?" or "what's the minimum
   widget height in grid units?". Shipping reorder first lets those
   questions answer themselves through usage.
3. The migration path is real — when (if) Step 2 ships, the
   `dashboard_layouts` table swaps its `jsonb` shape and the
   `<SortableDashboard>` wrapper becomes a `<GridLayoutDashboard>`
   wrapper. Nothing in the widget contract has to change to support
   that swap.

**Alternative rejected — persist in a `user_preferences` JSON column.**
Considered tucking the layout into a single shared preferences row
(layout + future notification settings + future tour-state, all in one
`jsonb` blob). Rejected because:

1. The composition philosophy of this codebase — registry pattern for
   widgets, per-feature Redux slices, sub-phase scoping — biases
   toward "give each concern its own surface." A shared preferences
   blob couples concerns that have no business being coupled at
   read/write time.
2. Concurrency is sharper when each concern owns its own row.
   Updating a layout shouldn't have to round-trip through a notification-
   settings serializer.
3. The migration cost to break preferences out later (when a *second*
   concern wants its own row) is higher than the cost of starting
   with a dedicated table.

**Alternative rejected — `react-beautiful-dnd`.** It's the historically
common pick, but Atlassian has stepped away from active development.
Modern accessibility behaviors (screen-reader announcements, keyboard
drag, RTL support) come for free with dnd-kit and require add-ons or
forks with react-beautiful-dnd. New code, new library.

**Why this works.**

- **Layered against existing seams.** `Dashboard.tsx` is already thin:
  it reads the registry, filters by role, drops each widget into a
  `<Grid>` cell. The sortable wrapper plugs in where the `<Grid>` is
  today. The widget contract `{ id, title, audience, Component }`
  doesn't have to change for Step 1 — `id` is the only field the
  sortable layer reads.
- **Persisted state stays small.** Step 1's persisted shape is a
  single array of widget ids. That's compact, easy to validate, and
  trivially upgrades to a richer shape later (the migration is
  additive — new keys default; the `order` array stays meaningful).
- **It's a strong demo moment.** Drag-to-reorder is visually
  immediate, tells the "registry → per-user layout" story crisply,
  and is the kind of feature recruiters and engineering leaders click
  on a live demo to try first. Pulling it earlier than its strict
  dependency would suggest may be the right call for the showcase
  side of this project.

**Revisit if.**

1. Users (real users on the live demo, or hiring reviewers in
   walkthroughs) consistently ask for resize behavior — would
   accelerate Step 2 from "if asked" to "next sub-phase."
2. A widget lands that fundamentally can't share a row with another
   widget (e.g., a full-bleed chart that needs the full container
   width regardless of layout) — would push toward a richer layout
   shape (per-widget `fullBleed: true` flag) even before Step 2.
3. Per-user layout demand expands beyond the dashboard (e.g., a
   per-user case-list column order) — would suggest generalizing
   `dashboard_layouts` into a `ui_layouts` table keyed by surface,
   before that second consumer ships.
4. Accessibility audit findings on the live demo flag keyboard-drag
   ergonomics — would prompt a deliberate review of the dnd-kit
   keyboard sensor wiring before promoting drag-and-drop into the
   default-on path.

**Files this would touch on implementation (Step 1).**

*Backend*

- `backend/src/database/migrations/<timestamp>-dashboard-layouts.ts`
  — new table
- `backend/src/dashboard/` (new module) — controller + service + DTOs
  for the two endpoints
- `backend/src/dashboard/dashboard.controller.spec.ts` — 404-not-403
  cross-tenant coverage + happy-path PATCH coverage

*Frontend*

- `frontend/src/features/dashboard/types.ts` — no change for Step 1
  (the `Widget` contract is sufficient as-is)
- `frontend/src/features/dashboard/dashboardLayoutSlice.ts` — new
  slice owning hot state + merge rule + debounced persist
- `frontend/src/features/dashboard/SortableDashboard.tsx` — new
  wrapper component owning the dnd-kit context and drag handles
- `frontend/src/features/dashboard/SortableWidgetCard.tsx` — new
  per-widget wrapper exposing the drag handle in the card header
- `frontend/src/pages/Dashboard/Dashboard.tsx` — swap the
  `<Grid container>` for `<SortableDashboard>` and pass
  `visibleWidgets` + layout + persist callback through
- `frontend/src/features/dashboard/dashboardLayoutSlice.test.ts` —
  merge-rule coverage (added widgets, removed widgets, no saved
  layout, malformed saved layout)

---

## Pre-flight 2 — Admin section composition reuses the widget-registry primitive

**Date:** 2026-05-21 (forward-looking — drafted alongside the P2.1 spec; no
implementation work scheduled yet; expected sub-phase **P2.1**).

**Decision.** The admin section at `/admin` reuses the `Widget` contract
introduced for the engineer dashboard in P1.8 (`{ id, title, audience,
Component }`). Two parallel registries exist — `dashboardWidgets` (P1.8,
`frontend/src/features/dashboard/registry.ts`) and `adminWidgets`
(P2.1, `frontend/src/features/admin/registry.ts`) — sharing the
contract via a re-export from `@/features/dashboard/types`. The
`<AdminSection />` page mirrors the structure of `<Dashboard />`: a
welcome header followed by a responsive `<Grid>` of role-filtered
widgets pulled from the local registry.

**Alternative rejected — generic `<WidgetPage registry={...} />` shell.**
Tempting: both pages do the same thing structurally, so factor the
shell into a single component that takes a registry as a prop and a
header slot for whatever the page needs. Rejected because:

1. The two pages have meaningfully different headers in their *future*
   shape. Admin's header will grow to carry tenant-wide stats and
   operator-facing alerts; Engineer's stays minimal because the
   widgets themselves carry the user-scoped data. Premature
   abstraction at a layer where the two surfaces are about to diverge.
2. The `data-tour-id` namespaces are different (`dashboard.*` vs
   `admin.*`) — encoding that into a generic shell forces either
   accepting a `tourPrefix` prop or hardcoding it, both of which add
   parameters to a primitive that doesn't yet have a third consumer.
3. Two near-copies of a ~30-line page shell is cheap. The cost of
   maintaining duplication is bounded; the cost of the wrong
   abstraction grows with each new page wanting a different shape.

**Alternative rejected — single combined `allWidgets` array with
role-only filtering.** Considered keeping one global registry where
each widget's `audience` field drives where it lands: `ENGINEER`/`ADMIN`
widgets render on `/`, `ADMIN`-only widgets render on `/admin`. Rejected
because:

1. Role isn't the only axis that matters. The engineer dashboard and
   the admin section answer different *questions* (mine vs. tenant's),
   not just have different audiences. Conflating the two registries
   invites accidental cross-audience widgets and obscures the
   architectural intent.
2. The registry-as-source-of-truth pattern depends on the registry
   being scoped to a single concept. Once a registry holds widgets
   for two unrelated surfaces, "where does this widget belong?"
   becomes a non-obvious question for every new widget.
3. Adopter friction. The dashboard README's "how to add a widget"
   instructions become more complex if the same registry serves two
   pages with different conventions.

**Why this works.**

- **Composition pattern proves itself by replication.** The
  widget-registry pattern earns its keep precisely when it shows up
  in a second surface without modification. P2.1 is that proof.
- **The contract is the abstraction; the page is the consumer.**
  Sharing `Widget` between two pages is the right reuse boundary.
  Sharing the page shell between two pages would be the wrong one.
- **Future surfaces stay open.** A P14 customer-facing dashboard
  could land as a third registry (`customerWidgets`) without changing
  any existing code. The pattern's growth path is additive.

**Revisit if.**

1. A third widget surface arrives (e.g., P14 customer dashboard) and
   the three pages are still structurally identical, with no
   divergent header treatment — that's the moment to extract a
   generic shell.
2. The `Widget` contract grows fields that only one registry uses
   (e.g., admin widgets want a `confirmDestructive: boolean`). At
   that point, either keep the field optional on the shared contract
   or split the contract per registry.
3. A widget genuinely belongs on both surfaces (e.g., a tenant-wide
   activity feed visible on both `/` and `/admin`). The registry
   pattern doesn't preclude listing the same `Widget` in two
   arrays — it would just be the first cross-registry consumer and
   worth a Mid-flight note when it happens.

---

## Pre-flight 3 — Admin routing as a single `/admin` page (not `/admin/*` sub-routes) for now

**Date:** 2026-05-21 (forward-looking — drafted alongside the P2.1 spec;
expected sub-phase **P2.1**).

**Decision.** `/admin` is a single route hosting all admin widgets,
not a section with sub-routes. The route map adds one entry:

```tsx
<Route element={<RequireRole role="ADMIN" />}>
  <Route path="/admin" element={<AdminSection />} />
</Route>
```

When (and only when) the admin widget count crosses ~5 and the page
becomes cognitively crowded, split into `/admin/users`,
`/admin/customers`, etc. — each with its own sub-page composed of a
subset of `adminWidgets` filtered by category.

**Alternative rejected — `/admin/*` sub-routes from day one.**
Considered structuring the admin surface as a section with its own
sub-routes from the first sub-phase: `/admin/users` (where
`PendingApprovalUsers` lives), `/admin/settings`, etc. Rejected
because:

1. With one widget, sub-routes are pure structural overhead. There's
   no navigation problem to solve.
2. Sub-routes require a navigation primitive (sidebar, tab bar, etc.)
   to be useful. Building that primitive for one destination is wasted
   work; building it for two is borderline; for five it's earned.
3. Premature URL commitment. The right sub-routes only become obvious
   once 3-4 admin widgets exist and natural groupings emerge.
   Committing now risks renaming later (which is fine in a private
   project but disrupts any external bookmarks or shared links).

**Alternative rejected — single-page tabs within `/admin`.** Considered
having `/admin` carry MUI `<Tabs>` between widget categories ("Users,"
"Customers," "Settings") from day one. Rejected because:

1. Same overhead problem as sub-routes but worse: the tabs are visible
   even with one tab's worth of content, which signals an empty
   surface.
2. Tabs and sub-routes solve overlapping problems; picking one early
   forecloses the other. Deferring lets the right structure emerge.

**Why this works.**

- **Adopter clarity.** A new admin widget's "where does this go?"
  question has one answer: `adminWidgets`. No category decision.
- **Cheap to split later.** When the split happens, it's purely
  additive: the existing `/admin` becomes either a landing page or
  redirects to the first sub-route. No URL renames; no removed routes.
- **Matches the engineer dashboard's evolution.** The engineer
  dashboard didn't ship with widget categories either. If the dashboard
  ever needs sub-routes, the admin split blueprint will already exist.

**Revisit if.**

1. The admin widget count crosses ~5 and the page becomes cognitively
   crowded — the natural moment to split.
2. A single widget grows complex enough to need its own URL (e.g., a
   user-detail drill-down at `/admin/users/:id`). That single sub-route
   can land without splitting the rest of the admin surface.
3. A future widget needs filter / sort state that should be
   bookmarkable. URL-based state on `/admin?widget=users&filter=...`
   gets unwieldy fast; that's another natural moment to split.

---

## Pre-flight 4 — Role gating as route wrapper (`<RequireRole>`) rather than runtime switch

**Date:** 2026-05-21 (forward-looking — drafted alongside the P2.1 spec;
expected sub-phase **P2.1**).

**Decision.** Introduce a new `<RequireRole role="ADMIN" />` outlet
wrapper in `frontend/src/routes/`, alongside the existing
`<RequireAuth />` and `<RequireActiveAuth />` wrappers. The wrapper
renders `<Outlet />` when the current user's role matches; otherwise
it redirects to `/` via `<Navigate to="/" replace />`. `/admin` lives
inside this wrapper in the route map.

**Alternative rejected — `<AdminRouter />` runtime switch mirroring
P1.8's `<DashboardRouter />`.** P1.8 introduced `<DashboardRouter />`
to dispatch between `<Dashboard />` (for ENGINEER/ADMIN) and `<Home />`
(for CUSTOMER) at the `/` route. The pattern *could* extend to
`/admin`: a `<AdminRouter />` that renders `<AdminSection />` for ADMIN
and falls back to a 403 / redirect for everyone else. Rejected because
the two situations have different semantics:

- *`<DashboardRouter />` is a **switch**.* The `/` URL is meaningful
  for every authenticated role — the question is *what to render
  for whom*. Both ENGINEER and CUSTOMER pageloads route through the
  switch and get a real (different) page.
- *`/admin` is a **gate**.* The URL itself is admin-only. Non-admins
  shouldn't pageload through the admin code path at all. A runtime
  switch leaks the URL's existence (the route is registered; the
  component just decides not to render) and forces every non-admin
  request through admin-related render logic, which is wasteful and
  invites accidental information leakage if the gate component ever
  shares state with the gated content.

**Alternative rejected — checking role inline at the page component
(no wrapper).** Considered putting the role check inside
`<AdminSection />` itself — render the page if ADMIN, otherwise
`<Navigate to="/" />`. Rejected because:

1. The pattern doesn't compose. Every role-gated page would re-implement
   the same check. A wrapper centralizes the policy and ensures
   consistent redirect behavior.
2. It mixes concerns. Page components are about *what to render*;
   route wrappers are about *who can render this URL*. Keeping the
   layers distinct keeps each layer simpler.
3. The existing `<RequireAuth />` / `<RequireActiveAuth />` wrappers
   set the convention. Following the same pattern for role gating is
   the path of least surprise.

**Why this works.**

- **Composes with existing route wrappers.** The route map gets a
  natural nesting: `<RequireActiveAuth />` (must be ACTIVE) →
  `<Layout />` (chrome) → `<RequireRole />` (must be ADMIN for this
  branch) → page. Each layer enforces one concern.
- **Reusable.** Any future role-gated route uses the same wrapper.
  When P14's customer-facing surfaces need CUSTOMER-only routes,
  `<RequireRole role="CUSTOMER" />` works without modification.
- **Easier to test.** The wrapper is a small component with a clear
  contract; a focused unit test covers "renders Outlet when role
  matches" and "redirects when it doesn't." No need to test role
  gating inside every gated page.

**Revisit if.**

1. The pattern proves awkward when role-gated routes need different
   fallbacks (some redirect to `/`, some to a 404, some to a "request
   access" page). At that point, the wrapper grows a `fallback?:
   ReactNode` prop or splits into named variants.
2. Multi-role gates become common (e.g., "ADMIN or ENGINEER may view
   this"). The wrapper signature would extend to `roles: UserRole[]`
   — straightforward additive change but worth a Mid-flight when
   it lands.
3. The redirect target needs to vary by source URL (e.g., a deep-linked
   admin URL that should round-trip after login). At that point, the
   wrapper captures the source URL into router state and the post-login
   flow restores it. That's its own decision and lands when the
   feature is asked for.

---

## Pre-flight 5 — Email transport: Resend now, SES at P12 prod hardening

**Date:** 2026-05-23 (P2.2 spec drafting — locking in transport choice before the implementation lands).

**Decision.** Use [Resend](https://resend.com) as the outbound email transport for the entire P2.2 lifecycle (admin registration alert, customer welcome, customer approval-granted). SDK: `resend@^4.0.0`. From-address must match a verified Resend domain — `caseflow.musto.io` becomes the production-ish from-address (`noreply@caseflow.musto.io`) once the one-time DNS verification completes; the sandbox sender (`onboarding@resend.dev`) is acceptable for the initial wiring smoke. SES becomes the prod transport at P12 prod hardening; the `EmailTransport` interface already supports the future SES adapter as a drop-in.

**Why Resend over SES.**

- *Setup cost.* Resend is sign-up + API-key + (optional) DNS verification — ~4 hours end-to-end. SES is sandbox application + DNS verification + sandbox-out-of-sandbox approval + bounce-handler wiring + suppression-list config — ~14 hours, much of it waiting on AWS approval. For a portfolio MVP, the SES setup tax is not defensibly justified.
- *Free tier headroom.* Resend's free tier is 3,000 emails/month and 100/day. Portfolio demo traffic sits orders of magnitude below those caps. The cost ceiling is high enough that exceeding it would itself be a strong signal worth a Mid-flight (audience growth needing real attention).
- *Reliability for the use case.* Resend's deliverability for transactional email is competitive with SES at the scales we care about. The deliverability gap that justifies SES doesn't materialize until you're sending millions/month or need enterprise-grade bounce + complaint pipelines — both P12 concerns.
- *In-AWS bonus is real but premature.* SES's IAM-native auth is genuinely nicer than API-key-via-Secrets-Manager. But the project already has the Secrets Manager path wired (`JWT_SECRET`, `DATABASE_URL`) from P10 — adding `RESEND_API_KEY` alongside is one new secret key, not new infrastructure.

**Alternative rejected — SES from day one.** Production-grade and IAM-native. Rejected for the setup cost above. Hard rule: SES is the prod transport, Resend stays the dev transport, the swap happens once at P12 — explicit decision boundary, not a deferred-indefinitely "maybe later" handwave.

**Alternative rejected — both adapters now, env-selected.** Builds twice the code for half the value. The current `EmailTransport` interface (`send(message): Promise<void>`) already supports the future SES adapter as a drop-in; nothing about Resend-first prevents it. Building both up-front would also force a third decision (which to pick by default) and create an untested code path until SES is exercised in real traffic.

**Alternative rejected — SMTP via Postmark / Mailgun / SendGrid.** Considered briefly. Postmark's transactional-only positioning is defensible, but the API surface is the same as Resend with no operational advantage at our scale, and Postmark requires a paid plan from day one. Mailgun has historical deliverability issues for low-volume senders. SendGrid's free tier is fine but the SDK ergonomics are worse. Resend wins on the combined axis of setup cost + free-tier generosity + SDK quality.

**Revisit if.**

1. Resend reliability drops below acceptable for the demo audience (>1% delivery failures observed in dev logs over a one-week window). At that point, trigger the early SES swap.
2. The project crosses Resend's free-tier limits — same trigger.
3. A specific enterprise demo viewer requires a transport choice that supports their domain's DMARC `p=reject` posture and Resend's deliverability profile isn't sufficient.

**Cross-reference.** [`p2-2-admin-notification.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-2-admin-notification.md) "Resolved pre-flight decisions" section; P12 phase entry in [ROADMAP.md](../../ROADMAP.md) (production hardening); future SES adapter file `backend/src/notifications/transport/ses.transport.ts` (lands at P12).

---

## Pre-flight 6 — All three transactional email types ship together in P2.2

**Date:** 2026-05-23 (P2.2 spec drafting — scope-resolution call).

**Decision.** P2.2 ships three email types in one sub-phase: admin registration alert (the original P2.2 scope from the ROADMAP), customer welcome on registration, and customer approval-granted on `PATCH /users/:id/status` PENDING_APPROVAL → ACTIVE transition. Defer-customer-emails was the original lean in the ROADMAP; this Pre-flight reverses it.

**Why ship all three together.**

- *Demo narrative integrity.* The original ROADMAP open call was "decide based on demand from the LinkedIn audience trying the register-and-approve demo flow." Evaluating that demand requires the register-and-approve flow to be conversational — admin-alert-only leaves the registrant in silence between submitting the form and getting approved, which is the exact moment of highest drop-off risk for the demo. Three emails together close the loop end-to-end.
- *Cost shape.* Ship cost is ~2× admin-alert-alone (template work doubles, integration work is identical, factory + transport are already there). Shipping them together avoids a follow-up sub-phase that would otherwise sit on the backlog through three later phases waiting for demand-evaluation evidence that never lands until the loop is actually closed.
- *Trust signal for the portfolio audience.* The customer-welcome + approval-granted pair signals "this product takes user experience seriously" without doing any extra work beyond the templates. Conversely, admin-alone signals "we did the engineer-side plumbing and stopped" — accurate but understates the product thinking.

**Why not defer welcome + approval to a post-P2.2 evaluation.** The "evaluate-then-ship" loop is a textbook over-optimization. The cost of building two more templates (~3 hours including the spec + tests) is lower than the cost of running an evaluation + capturing results + spec-drafting the follow-up + re-loading context. The asymmetry favors shipping.

**Alternative rejected — admin alert only, customer emails deferred.** Tempting for speed. Rejected because it creates a silent-system gap right where the demo's narrative ("we approve responsibly") wants to be loudest. The cost saved is real but the cost incurred (a half-finished story for demo viewers) is worse.

**Alternative rejected — admin alert + welcome only, approval-granted deferred.** Halfway-house option. Rejected because approval-granted is the highest-value of the three — the moment a customer learns they can use the product is the moment with the most product-trust impact. Dropping it would be the wrong half to drop.

**Revisit if.** Resend free-tier limits become a real constraint. At that point, dropping the welcome email is the lowest-value drop (customer already knows they registered, the email mostly confirms expectations). Approval-granted is the most-valuable retain.

**Cross-reference.** [`p2-2-admin-notification.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-2-admin-notification.md) "Resolved pre-flight decisions" + "Acceptance criteria"; Pre-flight 5 (Resend transport supports all three with no per-type cost).

---

## Pre-flight 7 — Direct injection + non-blocking dispatch; no event bus, no outbox

**Date:** 2026-05-23 (P2.2 spec drafting — architectural call on the dispatch shape).

**Decision.** `NotificationsService` is injected directly into the call sites (`AuthService`, `UsersController`). Each call site uses `void` to fire without awaiting. All error handling lives inside `NotificationsService` — methods catch their own errors, log with structured context (user id, notification type, transport error), and resolve successfully. **Followed by Mid-flight 4** which adds `.catch(() => undefined)` as a defense-in-depth layer at every call site under Node 24's strict unhandled-rejection policy.

**Why direct injection + non-blocking dispatch.**

- *Three call sites is too few for indirection.* Event-emitter pub/sub sounds nice ("decouple AuthService from NotificationsService") but the decoupling claim is theoretical at this scale. Both modules ship in the same deployable artifact; the indirection only pays off when listeners multiply or the producer/consumer lifecycles diverge. Neither is true for three controllers calling one service.
- *Latency hiding without queue weight.* The HTTP response from `register()` or `updateStatus()` doesn't wait on email transport. `void` at the call site is the minimal mechanism that achieves this without dragging in a queue, worker, retry logic, or persisted outbox. The transport's success/failure surfaces only in logs (the audit trail), which P11 turns into alarm-able metrics when observability lands.
- *Failure isolation is the service's job, not the caller's.* If notification methods could leak rejections to callers, every call site would need its own try/catch — fragmenting the error-handling rule across the codebase. Centralizing it inside `NotificationsService` (single try/catch per method, single log shape, single audit context) is cleaner and harder to break.

**Alternative rejected — `@nestjs/event-emitter` pub/sub.** In-process events ("decouple AuthService from NotificationsService") sound architecturally cleaner but the decoupling claim is premature. Three call sites is too few to amortize the indirection cost. The "easier to add more listeners later" argument is real but speculative: P7 (AI foundation) will not extend P2.2's emitter; it will build a different mechanism (outbox → SQS) appropriate for its scale.

**Alternative rejected — outbox pattern (DB row written in same transaction as user-create, worker drains and sends).** Real-world correct for at-least-once delivery: write a `notifications_outbox` row inside the same transaction as `createUser`, then a worker process drains it. Rejected because P2.2 is portfolio-MVP scale. The cost (new table, worker process, retry semantics, observability for queue depth) does not earn the value (at-least-once guarantees) at this stage. P11 introduces observability that would make outbox visible; P12 introduces prod hardening that would make at-least-once material. P2.2's "fire and forget + log on failure" is defensible for the actual user volume.

**Alternative rejected — BullMQ / dedicated queue library.** Same scale-vs-cost argument as outbox. BullMQ is a fine choice when AI workers land in P7; it's overkill for one email transport.

**Revisit if.** The decision log accumulates evidence that emails are silently failing in production (deliverability reports, support requests, missing-email complaints from the audience). At that point, the outbox migration is well-scoped: `NotificationsService` stays the only public API; internally, the implementation changes from direct-call to enqueue.

**Cross-reference.** Mid-flight 4 (the runtime gap in bare-`void` under Node 24, fixed with paired `.catch()`); [`p2-2-admin-notification.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-2-admin-notification.md) Pre-flight 7 stub; P7.5 (SQS-backed AI processing — the natural next-scale dispatch pattern).

---

## Pre-flight 8 — Admin recipients: lookup, not env-var

**Date:** 2026-05-23 (P2.2 spec drafting — reversing the ROADMAP's tentative `ADMIN_NOTIFY_EMAIL` env-var lean).

**Decision.** `NotificationsService.notifyOfNewRegistration` resolves recipients dynamically by calling `UsersService.listAdminsByTenant(newUser.tenantId)`. Every ACTIVE admin in the new registrant's tenant gets a copy of the alert. **No `ADMIN_NOTIFY_EMAIL` env var.** The lookup adds one extra DB query per registration; the trade is "negligible runtime cost" vs "multi-tenant correctness by default."

**Why dynamic lookup over env-var.** Env-var recipient routing is correct exactly until a second tenant or a second admin exists. Both already exist in the seed harness (3 tenants, multiple admins). A single global env-var would:

1. Email Tenant A's admin when Tenant B has a new registration. Cross-tenant leakage of operational context, which is the same family of bug the tenant-scoped queries from P1.5 exist to prevent.
2. Force a future Mid-flight to refactor to per-tenant lookup the moment the demo audience tests with a second tenant. The Mid-flight would be exactly this decision being made under time pressure.
3. Mask "tenant has zero admins" as a config bug rather than a real bug (the env-var fallback would happily fire the alert to an unrelated admin, hiding the underlying schema invariant violation).

The lookup version handles all three correctly: every tenant routes its alerts to its own admins, the "zero admins" case logs a defensive warning rather than silently misfiring, and the multi-tenant invariant is preserved by construction.

**Why this is also the cleaner-codebase choice.** The lookup mirrors the existing tenant-scoped query pattern (`UsersService.listByTenant`, `CasesService.findAll`). A new env-var for recipient routing would create a precedent that says "this kind of cross-cutting config can live as a global env-var" — a precedent that gets harder to walk back when notifications expand to escalations (P9.2), SLA breaches (P5.3), etc. Each of those would arrive with its own env-var temptation if we accept one now.

**Alternative rejected — single global `ADMIN_NOTIFY_EMAIL` env-var.** What the ROADMAP originally proposed. Works for the single-admin-single-tenant demo profile, breaks the moment a second tenant signs up via the LinkedIn invitation. The fix would arrive as a Mid-flight — which is exactly the kind of architectural drift this decision log exists to prevent.

**Alternative rejected — per-tenant config table (`tenant_notification_settings`).** The right answer eventually for per-admin preferences (mute/unmute, digest mode, severity thresholds). Premature in P2.2. Add when a real preference emerges; the lookup-by-role-and-status query is the foundation that table will sit on top of.

**Alternative rejected — env-var fallback for the "zero admins" case.** Considered using `ADMIN_NOTIFY_EMAIL` as the fallback when `listAdminsByTenant` returns empty. Rejected because the env-var fallback would mask a real bug. If a tenant has zero ACTIVE admins, that's a schema-invariant violation (or seed-data bug) worth surfacing loudly, not a routing problem to paper over. The "zero admins" branch logs a warning and returns; the absence of the email is the diagnostic signal.

**Revisit if.**

1. Admin count per tenant grows past ~5. At that point, fan-out becomes noisy. A `notification_preferences` table (per-admin opt-in/out) earns its keep, and the lookup query becomes "ACTIVE admins where notifications.new_registration = true."
2. A real per-tenant notification preference emerges from the demo audience (e.g., "mute Tenant X's alerts during off-hours"). Same table, slightly extended schema.
3. Email-in case creation (P13) needs the same routing logic — sender-domain-to-tenant + tenant-to-admins. The query is the same shape; might warrant lifting into a `NotificationRecipientResolver` service if the two call sites diverge subtly.

**Cross-reference.** Pre-flight 12 in [`phase-01-multi-tenancy-auth.md`](phase-01-multi-tenancy-auth.md) (404-not-403 cross-tenant safety, same family of invariant); P1.5 tenant-scoped queries (the existing pattern this Pre-flight extends); [`p2-2-admin-notification.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-2-admin-notification.md) Pre-flight 8 stub.

---

## Mid-flight 1 — `TextEncoder` polyfill in `jest.setup.ts` for react-router 6.4+ tests

**Date:** 2026-05-22 (P2.1 implementation — first frontend test run that imports `react-router-dom`).

**Symptom.** `frontend/src/routes/RequireRole.test.tsx` failed to load with `ReferenceError: TextEncoder is not defined`. Stack trace pointed into `node_modules/react-router/dist/development/index.js:348:31`, triggered by the test's `import` of `react-router-dom`. The other two new P2.1 test files (`usersSlice.test.ts`, `registry.test.ts`) passed — neither touches react-router.

**Root cause.** Two facts collide.

- `react-router` v6.4+ instantiates `new TextEncoder()` at module-load time as part of its internal hashing/path handling.
- The `jest-environment-jsdom` we use does not expose Node's built-in `TextEncoder` on `globalThis`. jsdom builds its own window-shaped global and Node's `util.TextEncoder` lives on a different scope, so the reference resolves against jsdom's globals → `ReferenceError`. The mismatch is well-documented but only surfaces when a component test imports react-router-dom.

**Fix.** Two-line polyfill added to `frontend/jest.setup.ts`:

```ts
import { TextDecoder, TextEncoder } from 'util';

if (typeof globalThis.TextEncoder === 'undefined') {
  globalThis.TextEncoder = TextEncoder;
}
if (typeof globalThis.TextDecoder === 'undefined') {
  (globalThis as unknown as { TextDecoder: typeof TextDecoder }).TextDecoder = TextDecoder;
}
```

`TextDecoder` is added at the same time even though only `TextEncoder` blew up — the two travel together, and once one piece of code in react-router or a downstream dependency reaches for `TextDecoder` it would fail the same way. Pre-empt it.

**Why not the alternatives.**

- *`testEnvironmentOptions: { customExportConditions: ['node', 'node-addons'] }` in jest config.* Works in some setups, but flips the Node export-condition resolution which can pull in CJS variants of ESM packages and cause unrelated breakage. The polyfill is targeted and reversible.
- *Mock `react-router-dom`.* Would defeat the point of the test — we're verifying the route gate's interaction with the router, not just the component's render shape.
- *Switch to vitest.* Larger blast radius; the project's jest setup is stable. File this in the "if we ever migrate" bucket.

**Going forward.** Every future frontend test that imports from `react-router-dom`, `react-router`, or any library that transitively reaches for `TextEncoder` (some testing-library helpers, `msw`, certain `@mui` paths under specific configs) loads cleanly. No per-test imports required. New tests don't need to know this footgun exists.

**Cost.** Zero in production — `jest.setup.ts` only runs during test runs. Polyfill is a node-builtin import; no new dependency.

**Cross-reference.** Same shape as P1's Mid-flight 9 (Vite proxy + CloudFront behavior allowlist) and Mid-flight 10 (RTK thunk dedup via `condition`, not body) — small, easy-to-rediscover footguns made explicit so a future contributor doesn't burn an hour on them.

---

## Mid-flight 2 — Backend ESLint config repaired (Node + Jest globals, base rule conflicts)

**Date:** 2026-05-22 (P2.1 implementation — first `yarn lint` run on backend in some time).

**Symptom.** `yarn lint` against the backend workspace reported **457 errors across 25 files**, all in three categories: `no-undef` on `process` / `Buffer` / `__dirname` (Node globals), `no-undef` on `describe` / `it` / `expect` / `jest` / `beforeEach` / `fail` (Jest globals in `*.spec.ts` files), and `no-unused-vars` on NestJS DI constructor parameters (`constructor(private readonly users: UsersService)`), TS enum members, and intentionally-discarded destructured values like `const { passwordHash: _omit, ...rest } = user`.

**Root cause.** `backend/eslint.config.mjs` was missing four things the frontend config already had:

1. No `globals.node` in source-file `languageOptions` → every Node built-in failed `no-undef`.
2. No `*.spec.ts` override with `globals.jest` → every Jest global failed `no-undef` in every test file.
3. Base `no-undef` and base `no-unused-vars` (inherited from `eslint.configs.recommended`) were active alongside the TS-aware `@typescript-eslint/no-unused-vars`. The base rules don't understand TS — they misfire on parameter properties (the DI pattern's whole point) and on enum members (declared values are externally consumed by definition).
4. Only `argsIgnorePattern: '^_'` configured — no `varsIgnorePattern: '^_'`. The `_omit` destructured-discard convention failed even the TS-aware rule.

The errors were almost entirely **pre-existing** — they predate P2.1, predate Phase 1 close-out, and likely predate the ESLint 9 migration (the flat-config rewrite is a common moment for environment declarations to get dropped). They sat unread because nobody ran `yarn lint` in normal workflow.

**Fix.** Repaired `backend/eslint.config.mjs` with three overrides:

1. **Source TS files** — `globals.node + globals.es2022`. Disabled base `no-undef` and base `no-unused-vars`. `@typescript-eslint/no-unused-vars` configured with both `argsIgnorePattern: '^_'` and `varsIgnorePattern: '^_'`.
2. **Spec/test files** — adds `globals.jest`. Loosens `@typescript-eslint/no-explicit-any` since test doubles legitimately reach for `any`.
3. **Ignores** — explicit ignore list for `dist`, `build`, `coverage`, `node_modules`.

Added `globals` to backend devDependencies (already present in frontend).

**Why this is bigger than a lint cleanup.**

- The contributor-framework planning doc ([`docs/planning/contributor-framework.md`](../planning/contributor-framework.md)) §4 plans a `verify.yml` CI job that runs `yarn lint`. With a broken config, the gate would be useless — 457 errors of noise hides any real regression.
- The same doc's §3 plans a husky `pre-commit` hook running `yarn lint` always-run (Decision D3). With the old config, every commit would be blocked forever.
- The signal-to-noise principle: when lint always reports 457 errors, the 458th — a real bug — disappears. Functioning lint is a precondition for the contributor framework's enforcement strategy to work.

**Going forward.**

- Every workspace's `eslint.config.mjs` needs to declare both `globals.node` for build-tooling files (configs, CLI scripts) and `globals.jest` for `*.spec.ts` files. This applies to any future workspace (client app in P14, etc.).
- Mirrors P1's Mid-flight 11 (CloudFront path patterns `/resource*` vs `/resource/*`) in shape: a small config detail that has outsized downstream consequences.
- When the contributor framework promotes to dev-ready, its first chunk should include "run `yarn lint` on a clean checkout and confirm zero errors" as a precondition before husky + verify.yml land.

**Cross-reference.** P1 Mid-flight 9, P1 Mid-flight 11, [`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §3-§4.

---

## Mid-flight 3 — Frontend ESLint cleanup + `RequireRole` prop rename

**Date:** 2026-05-22 (P2.1 close-out — first `yarn lint` run on frontend after Mid-flight 2 made the backend config functional and prompted a symmetric pass on the frontend).

**Symptom.** `yarn workspace @caseflow-ai/frontend lint` reported **27 problems — 11 errors and 16 warnings** across the workspace, in four distinct categories:

1. **16 warnings — `simple-import-sort/imports` and `simple-import-sort/exports`.** Cosmetic. Triggered by the new P2.1 files (`admin/`, `usersSlice`, the test files) sorting imports differently than the rule's expected order, and by older files that had drifted since the rule was added.
2. **2 errors — `no-undef` on `ImportMetaEnv`** in `frontend/src/utils/env.ts:14,23`. The Vite ambient type.
3. **1 error — `jsx-a11y/aria-role`** on `frontend/src/App.tsx:92` (`<RequireRole role="ADMIN" />`). The rule reads the literal string `"ADMIN"` as a value for the HTML/ARIA `role` attribute and fails it as not a valid ARIA role.
4. **7 errors — `react/no-unescaped-entities`** across `QuickActions.tsx`, `AwaitingApprovalPage.tsx`, `Home.tsx`, `RegisterPage.tsx`, `NotFound.tsx`. All apostrophes in JSX prose text (`Don't`, `it's`, `you're`).

**Root cause.** Three distinct issues, plus one cosmetic one.

- **`no-undef` on `ImportMetaEnv`** — exact same root cause as Mid-flight 2 on the backend. The base `no-undef` rule (inherited from `js.configs.recommended`) doesn't understand TypeScript ambient types. `ImportMetaEnv` is declared in `vite/client` and exists only in TS-land; the TS parser handles it correctly, but the base rule sees an unknown identifier and trips. The frontend config simply hadn't been touched in the same pass as the backend repair earlier in the day.
- **`jsx-a11y/aria-role` on `RequireRole`** — false positive on a custom React component. The rule can't tell that `<RequireRole>` is a custom component and `role="ADMIN"` is a custom prop, not an HTML `role` attribute on a DOM element. The rule sees `role="ADMIN"` literally and asks "is `ADMIN` a valid ARIA role?" (no — valid ARIA roles are `button`, `navigation`, `dialog`, etc.). The rule's behavior here is a design choice — it errs toward checking all `role=` occurrences rather than risk missing a real ARIA violation on a custom component that wraps a DOM element.
- **`react/no-unescaped-entities`** — a historical pedantry rule. Originated when older JSX parsers had quirks with raw quote characters; modern React (16+) handles raw apostrophes in JSX text without issue. The rule still catches stray quotes that might be unintended HTML, but in practice for prose text inside `<Typography>` / `<p>` / `<span>` it just clutters source with `&apos;` for zero safety gain. Most modern style guides disable it.
- **`simple-import-sort` warnings** — pure import-order drift. Eligible for `--fix`.

The errors were almost entirely **pre-existing** — they predate P2.1, but only surfaced now because Mid-flight 2's symmetric backend repair prompted running `yarn lint` on the frontend too. Same "nobody had run lint in normal workflow" pattern as Mid-flight 2.

**Fix.** Four steps, all small.

1. **Autofix the import-sort warnings.** `yarn workspace @caseflow-ai/frontend lint --fix` — 16 warnings cleared, touched ~15 files with mechanical import re-ordering.

2. **Frontend `eslint.config.mjs` — disable two rules.** Mirrors the backend Mid-flight 2 pattern:

   ```js
   // Base rules disabled — TS-aware equivalents handle the same checks.
   'no-undef': 'off',
   'react/no-unescaped-entities': 'off',
   ```

3. **Rename `RequireRole`'s `role` prop to `requireRole`** in `frontend/src/routes/RequireRole.tsx` (interface + destructure + comparison) and update the single call site in `frontend/src/App.tsx`. Eliminates the HTML-attribute ambiguity that triggered `jsx-a11y/aria-role` without disabling the rule wholesale — the rule continues to catch real ARIA-role mistakes on actual DOM elements.

4. (No new dependencies needed — `globals` already present in frontend devDependencies.)

After: `yarn workspace @caseflow-ai/frontend lint` exits clean.

**Why prop-rename and not rule-config.** `jsx-a11y/aria-role` supports an `ignoreNonDOM: true` schema option that would have silenced this case without renaming. Rejected because:

1. The rename improves the call site (`<RequireRole requireRole="ADMIN" />` is unambiguous; the original `<RequireRole role="ADMIN" />` reads close enough to an HTML `role` attribute to confuse a future contributor too, not just the linter).
2. Disabling the rule wholesale loses real-bug coverage on actual DOM elements. The rule does correctly flag `<div role="navbar">` or `<span role="admin">` — both of which are real mistakes worth catching.
3. The rename is a five-minute change to two files, all pre-merge. Cost is bounded; the lint-config approach trades a recurring readability tax for a one-time edit.

**Why turn off `react/no-unescaped-entities` instead of escaping the apostrophes.** Two reasons.

1. The rule catches nothing that matters. Raw apostrophes in JSX prose render correctly across every browser; the rule's original safety case (older JSX parsers) is no longer real. The trade-off is "noisy lint" vs "real-bug coverage" — there's no real-bug coverage on the table.
2. Escaping the apostrophes pollutes seven lines of human-readable prose with `&apos;` for no UX gain. The diff cost is borne by every future reader of those files.

This is the same shape as the backend Mid-flight 2 disables of base `no-undef` / `no-unused-vars` — turning off rules that misfire on legitimate code is a defensible choice when the TS-aware (or, in this case, modern-React-aware) equivalents either don't exist or aren't needed.

**Why this is bigger than a lint cleanup.**

- Same contributor-framework readiness argument as Mid-flight 2. The planned `verify.yml` job ([`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §4) will run `yarn lint` against the whole monorepo — backend AND frontend. A 27-error frontend lint output makes the gate worthless on the frontend side, same as it would on the backend.
- The husky pre-commit hook ([`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §3, Decision D3) runs lint always. With a broken frontend config, every commit touching frontend files would be blocked — same blocker shape as the backend pre-repair state.
- Establishes a symmetric rule-disable pattern across backend and frontend. When future workspaces land (client app in P14), the eslint.config.mjs template is now consistent — disable base `no-undef` and base unused-vars wherever TS is the parser; disable `react/no-unescaped-entities` wherever React JSX is the output.

**Going forward.**

- Forward-looking Pre-flights 3 and 4 in this log reference the old `<RequireRole role="ADMIN" />` API. They're intentionally left as written — pre-flights are a record of the *design* at the time of writing; the implementation evolution lives in mid-flights. A future reader reading top-to-bottom sees the design call (Pre-flight 4), the rename rationale (this Mid-flight), and the current API (`requireRole`) in that order.
- The prop-name lesson generalizes: any future custom React component with a `role` prop should anticipate the same false positive and use a more specific name from day one (`requireRole`, `displayRole`, `tourRole`, etc.). Adding it to the contributor framework's "review checklist when adding new components" section is a worthwhile follow-up when that doc promotes.
- ESLint configs across the monorepo should stay in lockstep on the base-rule disables (`no-undef`, `no-unused-vars`). A workspace-level config drifting from the others is the failure mode that produced both Mid-flight 2 and Mid-flight 3 in the same day.

**Cross-reference.** Mid-flight 2 (same pattern, backend side, same day), [`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §3-§4, Pre-flight 4 (the `RequireRole` design call that this mid-flight inherits the API from).

---

## Mid-flight 4 — Fire-and-forget pattern: `void promise.catch(() => undefined)` instead of bare `void promise`

**Date:** 2026-05-23 (P2.2 Chunk B implementation — first backend test run after wiring `NotificationsService` into `AuthService.register` and `UsersController.updateStatus`).

**Symptom.** One Jest test suite (`auth.service.spec.ts`) failed under Node 24 (`v24.8.0`) with:

```
Jest worker encountered 4 child process exceptions, exceeding retry limit
```

The worker crashed at the test:

```ts
it('register resolves with the user payload even if notifications reject', async () => {
  notifications.notifyOfNewRegistration.mockRejectedValue(new Error('transport down'));
  notifications.notifyCustomerOfRegistration.mockRejectedValue(new Error('transport down'));
  // ... register call expected to still resolve
});
```

The other 16 suites and 113 tests passed. The notification-related Pre-flights (7, 8) had locked in fire-and-forget dispatch via `void this.notifications.notifyOfNewRegistration(...)` at the call site, trusting `NotificationsService` to catch its own errors internally.

**Root cause.** Two facts collide under Node 24's strict unhandled-rejection policy.

- **The `void` operator silences TypeScript's `no-floating-promises` lint rule, but does NOT register a rejection handler with Node.** `void` discards the returned promise as a *value*; it does not attach `.catch()` to it. The promise is still floating from Node's perspective.
- **Node 24+ defaults `--unhandled-rejections=throw`** (was a deprecation warning in v15, became the default in later majors). When a promise rejects without a handler, Node throws — and inside a Jest worker, that crashes the worker process.

The test deliberately broke the `NotificationsService` internal-catch contract by making the mock reject. In production, this can't happen because the real `NotificationsService` catches internally (Pre-flight 7). But the test surfaced a real gap: the contract is a *runtime* guarantee, not a *type* guarantee — anyone removing a try/catch inside `NotificationsService` in the future would re-introduce this exact crash in production, not just in tests.

**Fix.** Replace bare `void promise` with `void promise.catch(() => undefined)` at every fire-and-forget call site. Two extra tokens; eliminates the class of bug.

```ts
// Before (Pre-flight 7 as originally written)
void this.notifications.notifyOfNewRegistration({ newUser });

// After (Mid-flight 4)
void this.notifications
  .notifyOfNewRegistration({ newUser })
  .catch(() => undefined);
```

Applied at all five P2.2 call sites:

- `backend/src/auth/auth.service.ts` — `register()` (2 calls), `googleSignIn()` JIT branch (2 calls).
- `backend/src/users/users.controller.ts` — `updateStatus()` (1 call).

After: `auth.service.spec.ts` passes, full suite runs green (126 → 130+ tests passing across 17 suites).

**Why the two-layer defense matters.** The `void` and `.catch()` do *different* jobs.

- *`void`* silences TypeScript's static lint check (`no-floating-promises`). Without it, the lint pipeline blocks. Without the lint check, an awaited promise creep into the call site would block the HTTP response on transport latency — exactly what Pre-flight 7 was designed to prevent.
- *`.catch(() => undefined)`* registers a runtime rejection handler with Node, so a rejected promise can never escape to the unhandledRejection event. This is the production-safety net.

Removing either layer reintroduces a specific failure mode. Keeping both is the correct shape under Node 24's policy.

**Why this is the project's new standard.** Three reasons.

1. **Forward-compatibility with Node's tightening rejection policy.** Node's trajectory is unambiguous: `--unhandled-rejections=throw` will keep getting stricter, not looser. Code written today should assume that any floating rejection eventually crashes the process.
2. **Defense-in-depth for service-contract changes.** The internal-catch guarantee inside `NotificationsService` is a piece of *code*, not a piece of *types*. A future refactor that removes (or moves, or accidentally narrows) the try/catch would break the guarantee silently. The call-site `.catch()` survives that change.
3. **Documents intent at the call site.** Bare `void` reads as "I'm choosing not to await this." `void promise.catch(() => undefined)` reads as "I'm choosing not to await this AND I don't care if it rejects." The intent is now legible at the location it matters.

**Generalization — when to apply.** Anywhere a side-effect-with-no-return-value is fired without `await`, AND the underlying promise has any path that can reject. This includes:

- Notification dispatches (this Mid-flight).
- Future audit-log writes from the AI provider invocations (P7.4) when those happen out-of-band.
- Future webhook dispatches (P11 observability hooks, P13 email-in webhook acknowledgments).
- Any "fire telemetry off-thread" pattern where blocking the request is unacceptable.

Anywhere the upstream promise is guaranteed by its type signature to resolve (e.g., a pure-computation `Promise<T>` with no reject path), bare `void` is technically sufficient — but applying the `.catch()` uniformly is the simpler rule to remember and review.

**What this isn't.** This is not a substitute for proper error handling. The `.catch(() => undefined)` *swallows* the error at the call site. The error is captured (and structured-logged) inside `NotificationsService` before it ever reaches the `.catch()`. If a future fire-and-forget pattern is built without that internal log, the call site's swallowing would lose observability. The two layers work together; replacing one with the other breaks the audit trail.

**Going forward.**

- All future fire-and-forget call sites in this codebase use the paired pattern. New code review should flag bare `void promise` for floating side effects.
- The contributor framework ([`docs/planning/contributor-framework.md`](../planning/contributor-framework.md)) should add this pattern to the "review checklist when adding new async side effects" section when that doc promotes from planning to implementation.
- ESLint has a `@typescript-eslint/no-floating-promises` rule with an `ignoreVoid: true` default — it accepts bare `void`. Worth investigating a custom lint rule (or a stricter config) that requires the `.catch()` chain too, but that's a follow-up; the immediate fix is the pattern + this Mid-flight as the project's reference.
- The Node 24 strict-rejection policy applies equally to the frontend's Vite/Vitest stack — if any frontend fire-and-forget pattern emerges (e.g., a Redux thunk dispatched without awaiting), the same `.catch()` layer applies.

**Cross-reference.** Pre-flight 7 (direct injection + non-blocking dispatch — this Mid-flight is its production-safety completion); Node docs on `--unhandled-rejections=throw`; future custom lint rule TBD.

---

## How this file is updated

1. Pre-flight decisions are written before any sub-phase code lands.
   Forward-looking pre-flights (drafted well before their sub-phase
   opens) note that explicitly in the **Date** line.
2. New decisions discovered mid-phase get appended as `Mid-flight N —`
   sections with the same shape (Decision / Alternative rejected /
   Revisit if).
3. When P2 closes (sub-phases all ✅), this log's status flips to
   **Closed** and a one-line summary lands in the ROADMAP "Where to
   look" cell for P2.
4. Subsequent phases get their own `phase-NN-<slug>.md` log.
