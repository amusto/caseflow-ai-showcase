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

## Pre-flight 9 — Customer module structure: one `CustomersModule` housing both entities

**Date:** 2026-05-24 (P2.3 spec drafting — locking in the module shape before the entities land).

**Decision.** `CustomerOrganization` and `CustomerContact` live in a single feature module at `backend/src/customers/`. The module's `TypeOrmModule.forFeature([CustomerOrganization, CustomerContact])` registers both; P2.4's `CustomersService` + `CustomersController` will own both entities' CRUD; P2.7's frontend `features/customers/` mirrors the same coupling. P2.3 ships the module with empty `providers: []` and `controllers: []` arrays — the entities + the module shell only — because the controller and service work belongs in P2.4 where it can be reviewed alongside its DTOs and tests.

**Why one module.** The two entities are parent/child and joined on nearly every query of interest:

- "list contacts for this organization"
- "list orgs with their primary contact denormalized" (well, that's covered by Pre-flight 10's denormalization, but the join would otherwise be required)
- "list contacts in this tenant" (joins both)
- "find the org that owns this contact's email" (P13 sender matching)

Two modules that need each other's services would force either a circular import (paired `forwardRef`, like the Users ↔ Notifications case in P2.2 — but with no benefit here) or a third "shared" module that holds both — which is the single-module decision with extra steps. Single module is the cleaner shape.

**Alternative rejected — split into `CustomerOrganizationsModule` + `CustomerContactsModule`.** Defensible on "one module per entity" grounds, but the cost shape is wrong: every cross-entity feature pays an indirection tax, and the architectural story ("our customer-side data model") fragments across two folders for zero gain.

**Alternative rejected — fold into `UsersModule`.** `User` and `CustomerContact` are both "people-ish" rows, but they answer different questions (authentication subject vs. customer-side correspondent) and have different lifecycle, retention, and PII concerns. Folding them invites conflation. Separate modules with clear boundaries beats one module with a mixed concept.

**Revisit if.** Either entity grows enough sub-concepts that splitting would aid reasoning. Plausible triggers: `CustomerOrganization` accumulating contracts, billing addresses, tags, SLA tiers, etc. The split would be additive — the existing entities stay where they are; new sub-entities land in new sub-modules that the existing one imports.

**Cross-reference.** [`p2-3-customer-entities.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-3-customer-entities.md) "Resolved pre-flight decisions"; existing `UsersModule` (similar single-module shape housing the auth-subject identity + its refresh tokens via `AuthModule` next-door).

---

## Pre-flight 10 — Primary contact: denormalized fields on `CustomerOrganization`, not an FK to `CustomerContact`

**Date:** 2026-05-24 (P2.3 spec drafting — the highest-value contested architectural call in P2.3).

**Decision.** `CustomerOrganization` carries `primaryContactName`, `primaryContactEmail`, and `primaryContactPhone` directly as nullable `varchar` columns. There is no `primaryContactId` FK pointing into `customer_contacts` in P2.3. The denormalized fields can be populated from any of: a user-typed value at org-create time (most common in P2.4's UI), an auto-fill from the first `CustomerContact` row, or an explicit "set as primary" action on a contact (P2.7-ish).

**Why denormalize.** Three reasons.

1. *Chicken-and-egg.* A customer org can plausibly exist before any individual contact is entered (sales handoff: "we just signed Acme; details to follow"). Requiring a `CustomerContact` row before the org can be saved adds awkward two-step creation flows or forces stub contacts that are worse than denormalized strings.
2. *List-view performance.* The admin widget and engineer list view show the org name + primary contact at a glance. Denormalization makes that a single-table scan; the FK alternative requires a left join on every list query. At P2.4's scale, the join is cheap — at P11's scale, denormalization stays cheap.
3. *Edit ergonomics.* Updating "the primary contact's email" via a contact row creates a write-amplification expectation (write to `customer_contacts`, propagate to `customer_organizations.primaryContactEmail`). Denormalization makes the two writes independent and lets the org's "primary contact" line carry a freely-typed value when the contact isn't a system-of-record entry yet.

**The cost we accept.** Drift risk: if a `CustomerContact` row's email changes, the `CustomerOrganization.primaryContactEmail` may go stale. Mitigation: P2.4's update flow exposes both as separately-editable fields. The "primary contact" line on the org is the canonical at-a-glance summary; the `customer_contacts` rows are the canonical correspondent list. They serve different purposes; drift between them is acceptable. The P2.3 seed factory reinforces this — it auto-fills the org's `primaryContact*` fields from each org's first contact at seed time, but those values are independent rows once persisted.

**Alternative rejected — `primaryContactId` UUID FK into `customer_contacts`.** Defers the at-a-glance display problem to a join, and creates the chicken-and-egg constraint above. Cleaner normalization at the cost of every consumer paying the join tax and the create-flow complexity. The FK approach also forces a "which `CustomerContact` is the primary?" UI decision into every contact-create form — premature commitment to a UX pattern P2.4 hasn't even drafted yet.

**Alternative rejected — JSON column with the full contact blob.** Considered briefly. Loses the type safety of typed columns and the indexability of `primaryContactEmail` — P13's email-in sender matching (per the ROADMAP) wants `CustomerContact.email` as the primary match; a queryable `primaryContactEmail` is also useful as a fallback when an inbound email doesn't match any individual contact but does match an org's primary address.

**Revisit if.** The "primary contact" notion grows from a single denormalized line into a richer concept (e.g., "primary billing contact," "primary technical contact," "primary escalation contact"). At that point, lifting into a proper FK + role table (`customer_organization_primary_contacts` with `role` enum) is the right move, and the denormalized columns either go away or become a cached "default contact" view. The denormalization is sized to the current product story — single line, single purpose; the lift to a richer model is well-bounded when the richer story emerges.

**Cross-reference.** [`p2-3-customer-entities.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-3-customer-entities.md) "Resolved pre-flight decisions" + "Schema" tables; P2.3 seed factory (`backend/src/seed/factories/customer-organizations.factory.ts`) demonstrates the "auto-fill from first contact" pattern; P13.3 (email-in sender matching) consumes both `CustomerContact.email` and the org's `primaryContactEmail` as fallback.

---

## Pre-flight 11 — `CustomerContact` email uniqueness scoped to the customer organization

**Date:** 2026-05-24 (P2.3 spec drafting — the grain decision for the uniqueness index).

**Decision.** `customer_contacts.email` is uniqueness-scoped via a composite unique index on `(customerOrganizationId, email)`. Two different customer orgs can each have a contact with the same email address (legitimate: a consultant working with multiple client orgs). One customer org cannot have the same email listed twice. Tenant scope is enforced separately via a denormalized `tenantId` column + `(tenantId)` index on `customer_contacts` — the composite unique is the uniqueness rule, the tenant index is the query-perf rule. They serve different purposes and don't conflict. Email is stored lowercased (P2.4 service-layer concern; same pattern as `users.email`), so the unique index is effectively case-insensitive without needing a `LOWER(email)` expression index.

**Why org-scoped (not tenant-scoped, not global).**

- *Global unique on `email`* would prevent the consultant case (same address, two different client orgs). It would also create cross-tenant collision potential — Tenant A's customer's email refuses to insert because Tenant B's customer happens to have the same address. That's wrong: the two tenants don't share a customer namespace, and a customer's email isn't a system-level identifier.
- *Tenant-scoped unique on `(tenantId, email)`* would prevent one of your tenant's customer orgs from listing a contact that also exists at another of the same tenant's customer orgs. Defensible but restrictive: a consulting tenant managing multiple end-client orgs would constantly hit this collision for shared contacts.
- *Org-scoped unique on `(customerOrganizationId, email)`* matches how contacts are mentally used: "this person represents Acme in dealings with us." Two Acme records with the same email is a duplicate-row bug; the same email at Acme and at Globex is two correspondents in two relationships.

**The denormalized `tenantId` column.** P2.3 follows the existing project pattern (`cases`, `users` all carry `tenantId` denormalized) — every tenant-scoped query is a single-table scan; no join through the parent org just to filter by tenant. The DB cannot enforce cross-row consistency natively: a contact with `tenantId = A` could be inserted under a `customerOrganizationId` whose parent has `tenantId = B`. P2.4's service layer enforces the invariant by reading the parent org's tenant before insert; P2.3 documents the invariant in the entity's class-level docstring so a future contributor doesn't bypass the service layer without realizing.

**Alternative rejected — global unique on `email`.** See above. Wrong scope.

**Alternative rejected — tenant-scoped unique `(tenantId, email)`.** See above. Defensible but restrictive.

**Alternative rejected — application-only uniqueness (no DB index, service layer checks).** The composite unique constraint is the safety net when application-layer logic is bypassed (direct DB writes, future migrations, seed scripts). At P12 prod hardening's audit pass, having the constraint at the schema layer makes the SOC II "we enforce uniqueness" story trivial to demonstrate; without it, the story leans entirely on app-layer correctness.

**Revisit if.** The email-in surface (P13.3) needs to match an inbound email to exactly one `CustomerContact` and the org-scoped unique allows ambiguity (the same address appears under two orgs in the same tenant). At that point, the match algorithm becomes "by email + most-recently-corresponded org" or similar; the index doesn't change but the resolution rule grows. The P2.3 seed factory currently keeps every fixture email globally unique to dodge the ambiguity until P13 has a forcing function.

**Cross-reference.** [`p2-3-customer-entities.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-3-customer-entities.md) "Resolved pre-flight decisions" + "Schema" tables; Pre-flight 12 in [`phase-01-multi-tenancy-auth.md`](phase-01-multi-tenancy-auth.md) (404-not-403 cross-tenant safety, same family of tenant-boundary invariant); P1.5 tenant-scoped queries (the existing pattern this Pre-flight extends); P13.3 sender matching (downstream consumer of this uniqueness rule).

---

## Mid-flight 5 — `Layout.tsx` wordmark becomes a "back to /" link

**Date:** 2026-05-24 (P2.3 implementation — small UX polish noticed during the Chunk B seed-extension review).

**Symptom.** With P2.1's Admin link in the AppBar, navigating between `/` and `/admin` was asymmetric: the Admin button took ADMINs forward to `/admin`, but there was no obvious "back to dashboard" affordance. The `{env.SITE_NAME}` wordmark in the AppBar's title slot was rendered as plain `<Typography>` text — not a link — so the conventional "click the brand to go home" pattern wasn't available.

**Fix.** Wrap the wordmark in a `<Button component={RouterLink} to="/" color="inherit">`:

```tsx
// frontend/src/components/Layout/Layout.tsx
<Typography variant="h6" component="h1" sx={{ flexGrow: 1 }}>
  <Button component={RouterLink} to="/" color="inherit">
    {env.SITE_NAME}
  </Button>
</Typography>
```

The wordmark now navigates back to `/` for every authenticated user. For ENGINEER + ADMIN the destination is the Engineer Dashboard (P1.8); for CUSTOMER it's the existing `<Home />` placeholder via `<DashboardRouter />` (P14 customer-app stretch will replace this with a real customer surface). The button inherits the AppBar's color via `color="inherit"`, so the visual treatment matches the existing surrounding "Admin" and "Sign out" buttons rather than introducing a third visual rhythm.

**Why this rises to a Mid-flight (not just a styling commit).** Three reasons.

1. *Navigation affordances are shared infrastructure.* `Layout.tsx` is rendered for every authenticated route — every future sub-page inherits this behavior whether it asks for it or not. Future contributors adding admin sub-pages (P2.4+) or customer-facing surfaces (P14) need to know the brand is a back-to-root affordance so they don't accidentally re-implement it elsewhere.
2. *The asymmetry is what made the fix obvious.* P2.1 added a forward-nav button (`Layout.tsx` Admin link); P2.3's customer-model work made the Admin section's growing surface visible (P2.4's customer widget lands next), which surfaces the missing back-nav. The Mid-flight captures the design pattern — "every forward-nav from the dashboard gets a paired back-nav, and the brand wordmark is the canonical back-to-`/` path" — so the next contributor adding a sub-route knows whether to repeat the pattern or rely on the brand.
3. *Conventional pattern made explicit.* "Click the brand to go home" is web-wide convention, but conventions only work when they're consistent. Documenting the call now (rather than letting it sit as an undocumented behavior) means the pattern survives styling refactors — anyone touching `Layout.tsx` later sees the link wrapper exists for a reason.

**Going forward.**

- Any future top-level navigation surface (P14 customer-app's customer-facing layout) inherits the same pattern: the brand wordmark is the home link. If a future product surface wants the wordmark to mean something else (e.g., a multi-product top-level switcher), that's its own Mid-flight.
- The `data-tour-id` namespace doesn't currently identify the wordmark link. If a future tour needs to anchor a step on "click the brand to go home," add `data-tour-id="layout.brand-link"` to the button at that time.
- `<RouterLink>` inside `<Button>` is the canonical "Button-styled link" pattern used elsewhere in `Layout.tsx` (the Admin button uses the same shape). Using a plain `<a>` would skip MUI's button styling; using `<Link>` without the button wrapper would skip the AppBar's button visual rhythm.

**What this isn't.** This is not a navigation refactor. The route map (`App.tsx`) doesn't change. The wrapper components (`RequireRole`, `RequireActiveAuth`) don't change. Only the AppBar's visual treatment of the wordmark — from plain text to a button-styled link — changes. The behavior is purely additive: clicking the wordmark used to do nothing; now it goes to `/`.

**Cross-reference.** P2.1 Layout.tsx (the Admin button — the forward-nav this Mid-flight pairs); Pre-flight 4 (route gating shape — the wordmark link still passes through `<RequireActiveAuth />` and `<RequireRole />` as defined there); `frontend/src/features/tours/README.md` (data-tour-id conventions, in case a future tour anchors on the brand).

---

## Pre-flight 12 — Endpoint shape: nested for parent-context operations, flat for individual contact operations

**Date:** 2026-05-24 (P2.4 spec drafting — the most contested architectural call in P2.4).

**Decision.** The customer-side REST surface is hybrid. Parent-context contact operations use nested URLs (`POST /customers/:orgId/contacts`, `GET /customers/:orgId/contacts`); individual contact operations use flat URLs (`GET /contacts/:contactId`, `PATCH /contacts/:contactId`). Org operations are all flat under `/customers/:orgId` since there's no parent context above the org (the tenant is implicit via JWT).

```
GET    /customers                                — list orgs in tenant
POST   /customers                                — create org (ADMIN only)
GET    /customers/:orgId                         — get one org
PATCH  /customers/:orgId                         — update org (ADMIN only)
GET    /customers/:orgId/contacts                — list contacts for org
POST   /customers/:orgId/contacts                — create contact (ADMIN only)
GET    /contacts/:contactId                      — get one contact
PATCH  /contacts/:contactId                      — update contact (ADMIN only)
```

**Why hybrid.** The URL should express the relationship when the relationship is informative (creating a contact requires knowing which org it belongs to — nested URL makes that explicit) and stay quiet when the relationship is redundant (fetching a contact by UUID doesn't need the parent context, since UUIDs are globally unique). The middle path matches well-trodden REST conventions in mature APIs (Stripe, GitHub) where parent / child resources land here for exactly this reason.

**Implementation note.** Two controllers, one service. `CustomersController` owns `/customers` + `/customers/:orgId/contacts`; `ContactsController` owns `/contacts/:contactId`. Both share `CustomersService` — Pre-flight 11's cross-row tenant invariant + composite-unique violation translation live in one place.

**Alternative rejected — fully-nested URLs (`/customers/:orgId/contacts/:contactId`).** Forces every contact endpoint to surface the parent context, including reads that don't need it. Worse, it duplicates information already encoded in the contact's UUID — a `PATCH /customers/wrong-org-id/contacts/correct-contact-id` is ambiguous: do we ignore the org id, or 404 on the mismatch? The flat individual endpoints sidestep this entirely.

**Alternative rejected — fully-flat URLs (`/contacts?customerOrganizationId=…` for list, `/contacts` for create with org id in body).** Hides the relationship from the URL. A reader can't tell from the path that contacts are scoped to orgs. The create-by-body shape also makes the "this contact belongs to this parent org" check feel like a special case rather than the obvious URL constraint.

**Revisit if.** P14 customer-app exposes a customer-facing contact-management surface where the customer hierarchy is different (e.g., contacts belong to multiple orgs through a junction table). At that point, the relationship is no longer "contact belongs to exactly one org" and the URL shape changes accordingly. No P2.4-level pressure to anticipate that.

**Cross-reference.** [`p2-4-customer-crud.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-4-customer-crud.md) "API surface" + "Resolved pre-flight decisions"; Pre-flight 11 (the cross-row tenant invariant the shared service enforces); P13.3 sender matching (consumer of the `GET /contacts/:contactId` flat endpoint).

---

## Pre-flight 13 — No DELETE endpoint in P2.4

**Date:** 2026-05-24 (P2.4 spec drafting — scope deferral with a well-bounded revisit trigger).

**Decision.** P2.4 ships no DELETE endpoint for either `CustomerOrganization` or `CustomerContact`. The canonical "we don't do business with them anymore" path is `PATCH /customers/:id { status: 'INACTIVE' }`. Contact removal is also deferred — if a contact is no longer relevant, the engineer-facing UI in P2.7 may surface a delete affordance gated behind confirmation, or the contact may simply be left in place. No production path requires hard delete in P2.4.

**Why defer.** Two reasons.

1. *Cases haven't FK'd to customer orgs yet.* P2.5 adds `Case.customerOrganizationId`. The right delete strategy depends on what that FK's `onDelete` constraint is, and that decision lives in P2.5's spec. Designing the delete endpoint before the FK exists invites a Mid-flight when P2.5 makes the call. Deferring is the cheaper move.
2. *Status-INACTIVE covers the only real lifecycle.* No portfolio-demo scenario actually requires hard delete. The status enum handles "remove from picker without losing history" — which is the universal answer for SaaS customer records (real CRMs almost never hard-delete customer rows for the same reason).

**Alternative rejected — Add DELETE in P2.4, refuse if cases exist.** Workable but premature. The case-FK exists only after P2.5; before then, the DELETE endpoint has no real check to perform beyond "does the org exist." Adding it now produces an endpoint that does the wrong thing for ~30 minutes of clock time during the P2.4 → P2.5 transition.

**Alternative rejected — Add DELETE in P2.4 with CASCADE.** The schema has `ON DELETE CASCADE` from `customer_contacts` to `customer_organizations`, so a hard delete of an org would automatically drop its contacts. Acceptable in isolation, but loses correspondent history with no obvious recovery path. Inconsistent with the project's "never tombstone" stance on User rows.

**Revisit if.** A real product need surfaces (e.g., GDPR right-to-erasure once P12 prod hardening puts compliance gates in scope). At that point, the DELETE story includes audit-log entries, case-FK cascade strategy, and probably a soft-delete `deletedAt` column. Today none of that infrastructure exists.

**Cross-reference.** [`p2-4-customer-crud.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-4-customer-crud.md) "Out of scope"; P2.5 (the FK that this decision waits on); P12 prod hardening (the natural home for GDPR / audit-log + delete semantics).

---

## Pre-flight 14 — Primary contact denormalized fields are NOT auto-synced from `CustomerContact` writes

**Date:** 2026-05-24 (P2.4 spec drafting — the runtime-binding decision Pre-flight 10 deferred to implementation).

**Decision.** When a `CustomerContact` row is created or updated, the parent `CustomerOrganization`'s `primaryContactName` / `primaryContactEmail` / `primaryContactPhone` fields are NOT automatically modified. The denormalized fields are independent — set at org-create time (optionally auto-filled from the form's first contact entry), changed via explicit `PATCH /customers/:orgId` updates, and otherwise left alone. Pre-flight 10 (P2.3) accepted this drift as a feature, not a bug; P2.4 implements that explicitly by *not* writing a sync trigger.

**Why no auto-sync.** Three reasons.

1. *Pre-flight 10 already settled the architectural call.* The denormalized fields exist to serve a different purpose than the `customer_contacts` rows. Auto-syncing would conflate the two — once the org's "primary contact" line becomes a cached view of the first contact, the only-set-at-create-time semantics evaporate and we've effectively built an FK with extra steps.
2. *Write amplification.* A contact update would have to propagate to potentially-stale org fields, requiring either a transaction or eventual-consistency machinery. Both are heavyweight for the actual product need.
3. *Reader expectations stay clear.* The org's primary contact is the canonical at-a-glance summary, intentionally separate from the canonical correspondent list (`customer_contacts`). Anyone needing one or the other reads the right table. Mixing them via auto-sync breaks the mental model.

**The "auto-fill from first contact" affordance.** The P2.7 create-org flow may offer a UI shortcut: when an engineer types a primary contact's name + email into the org form AND also adds them as the org's first contact, both surfaces should populate. That's a *one-shot fill at form submit*, not a runtime data binding. Same effect, fundamentally different semantics — and the right place for it is the UI layer, not the service layer. The P2.3 seed factory demonstrates the pattern (each org's primary fields are populated from its first contact spec at seed time, but the persisted rows are independent thereafter).

**Alternative rejected — Auto-sync via a service-layer trigger.** When `createContact` runs and the parent org has null primary fields, copy this contact's fields up. Sounds harmless but creates surprising behavior: the second contact added would also fail to populate (org already has primaries), creating an asymmetry that depends on insert order. Worse: deletes (if added later) would orphan the org's primary fields without a clear "promote a different contact" handoff.

**Alternative rejected — Auto-sync only when fields are null.** Same first-write-wins shape, slightly less surprising. Still loses to the "if you want the org's primary contact to track a contact row, use an FK" objection.

**Revisit if.** Pre-flight 10 is revisited (i.e., the denormalized fields are lifted into an FK + role table). At that point, the new model has explicit semantics for "this contact is the primary" and the question goes away. Until then, no sync.

**Cross-reference.** Pre-flight 10 (P2.3) — this Mid-flight is its runtime-binding completion; [`p2-4-customer-crud.md`](../../sdlc-roadmap-requirements/docs/tasks/dev-ready/p2-4-customer-crud.md) "Resolved pre-flight decisions"; P2.7 spec when it lands (the form-layer "auto-fill from first contact" affordance is a P2.7 implementation detail).

---

## Mid-flight 6 — Test-store reducer drift from production store

**Date:** 2026-05-24 (P2.4 Chunk B implementation — caught by `tsc --noEmit` after the new `customersReducer` landed in `src/store/index.ts`).

**Symptom.** Adding `customersReducer` to the production `configureStore({ reducer: { ... } })` map widened the global `RootState` type to include `customers: CustomersState`. Every Redux thunk in the codebase that closes over `state: RootState` (which is every `createAsyncThunk` with state injection) now expects any store it's dispatched against to carry `customers` in its state.

Two pre-existing test files broke as a result:

- `frontend/src/features/users/usersSlice.test.ts` (10 dispatch sites): `Argument of type 'AsyncThunkAction<…>' is not assignable to parameter of type 'UnknownAction'.`
- `frontend/src/features/admin/widgets/PendingApprovalUsers.test.tsx` (1 dispatch via render): same root error.

Both test files build a local `configureStore({ reducer: { auth: …, users: … } })` that omits `customers`. The thunks they dispatch (`fetchUsersThunk`, `updateUserStatusThunk`) require `state: RootState` — a state shape with `customers` in it. The local stores don't satisfy that shape, so TypeScript rejects the dispatch.

Two other test files were untouched because they don't dispatch RootState-constrained thunks: `RequireRole.test.tsx` (renders a route gate with `useAppSelector` only) and `TourEngine.test.tsx` (already includes the relevant slices).

**Root cause.** A reducer-map drift between the production store and per-test stores. When `RootState` is computed from the production store (`ReturnType<AppStore['getState']>`), every thunk inherits a "the state must look like this" constraint, but per-test stores can omit slices they don't exercise. Most of the time the omission is harmless — the thunk doesn't actually read the omitted slice. But the *type* doesn't know that, so TypeScript rejects the dispatch regardless.

**Fix.** Added `customers: customersReducer` to the two drifted test stores. Three lines of edit per file plus an import. The thunks dispatch cleanly and the tests behave identically — no runtime change in either suite.

**Why this rises to a Mid-flight.** The drift isn't a bug in either the production store or the test stores in isolation — both are internally consistent. The bug is in the *coupling*: the production store dictates `RootState`, and every test store has to mirror it. Future contributors adding a new slice will hit the same fail-to-typecheck pattern unless they know to update every test store too. The fix path (a) needs to be documented, and (b) is a candidate for the contributor-framework's planned `verify.yml` lint job — a check that every test-store reducer set is a superset of the production set (or at least a comment marker for intentional omissions).

**Going forward.**

- Any future sub-phase adding a new slice to `src/store/index.ts` must update every test store with a local `configureStore` to include the new reducer. The pattern is mechanical but easy to miss without `tsc --noEmit` as a gate.
- The contributor-framework's planned `verify.yml` ([`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §4) should add this check — either as a custom lint rule or via `tsc --noEmit` failing the CI on any test-store drift.
- A small `makeTestStore()` helper in `frontend/src/test-utils/` would centralize this: a function that constructs a store with all production reducers and accepts a `Partial<RootState>` preload. New contributors would use the helper by default. The helper deferred to its own follow-up sub-phase to keep this Mid-flight bounded — but tracked as a follow-up TODO.
- Same family as P1 Mid-flight 9 (proxy-allowlist drift between vite.config.ts and CloudFront) and Mid-flight 7 below — a recurring "two places agree by convention, not by contract" footgun.

**What this isn't.** This is not a Redux design problem. The `state: RootState` constraint is correct — thunks SHOULD know about the full state shape they can read from. The fix isn't to weaken the constraint; it's to make the test-store / production-store coupling explicit.

**Cross-reference.** P1 Mid-flight 9 (same family of drift, different surface); Mid-flight 7 below (same family, third occurrence inside ~2 weeks); [`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §4 (planned verify.yml hook that would catch this).

---

## Mid-flight 7 — Proxy / CDN allowlist drift (P1 Mid-flight 9 strikes again)

**Date:** 2026-05-24 (P2.4 Chunk B local smoke — admin page rendered blank, console showed `items.filter is not a function` at `customersSlice.ts`).

**Symptom.** After landing the P2.4 backend + frontend wiring, navigating to `localhost:8080/admin` produced a blank page and the console error `Uncaught TypeError: items.filter is not a function` at `selectActiveCustomerOrgs` in `customersSlice.ts`. The selector calls `items.filter(...)`, which requires `items` to be an array. The actual `items` value was the body of Vite's `index.html` fallback HTML response — not an array — because the dev server had no proxy entry for `/customers` and silently served the SPA bundle instead of forwarding to the NestJS backend.

**Root cause.** Identical to P1 **Mid-flight 9** (`/tour-state` proxy drift, 2026-05-19). Every new backend resource needs three touchpoints:

1. *NestJS controller* — the source of the route. P2.4 ships `CustomersController` at `/customers` + `ContactsController` at `/contacts`.
2. *Vite proxy entry* — `frontend/vite.config.ts` `server.proxy[...]`. Missing entry means Vite returns the SPA HTML on every path that doesn't match a known static asset.
3. *CloudFront ordered cache behavior* — `infra/modules/frontend_cdn/main.tf`. Missing behavior means the deployed dev environment serves S3's SPA fallback instead of forwarding to the ALB.

P2.4 added the controllers (item 1). Items 2 and 3 silently fell through. Locally, item 2 was the visible break; item 3 would only have surfaced when the admin page is loaded against the deployed `caseflow.musto.io` environment.

**Why the existing Mid-flight 9 warning didn't prevent it.** `vite.config.ts` already carries an inline comment block — `⚠ Allowlist drift gotcha (Mid-flight 9): every new backend resource prefix needs a line here AND a matching CloudFront behavior…`. The footgun still fired. That's the strongest possible signal that *comments are not a lint check*. The contributor-framework's planned `verify.yml` ([`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §4) is the real fix: a CI step that diffs every `@Controller('X')` in the NestJS codebase against the proxy + CloudFront allowlists and fails the build on mismatch.

**Fix.** Three touchpoints rectified.

1. **Vite proxy** — `/customers` added (Armando, during local smoke). `/contacts` added at close-out (no P2.4 frontend consumer yet — the engineer-facing contact UI lands in P2.7 — but landing the entry now means P2.7 doesn't have to rediscover the same footgun).
2. **CloudFront behaviors** — two new ordered behaviors in `infra/modules/frontend_cdn/main.tf`: `/customers*` and `/contacts*`. Pattern matches the `/users*` and `/tour-state*` precedent set by Mid-flight 11 in phase-01 (bare-resource matching requires `*` without leading slash, see P1 Mid-flight 11 for the diagnostic). Both behaviors land before P2.5 ships; the deployed dev environment needs a `terragrunt apply -auto-approve` in `infra/live/dev/frontend_cdn` to pick them up.
3. **This decision log entry** — making the third occurrence (P1.7 was the first against `/tour-state`, P2.4's `/customers` is the second, P2.4's `/contacts` is the third) visible enough that the contributor-framework promotion catches it.

**Why this Mid-flight, not just an amendment to P1 Mid-flight 9.** Mid-flight 9 documented the footgun. This Mid-flight documents that the footgun fires again under the existing mitigation (inline warning comments). The take-away has shifted from "be careful" to "automate the check." The two entries serve different purposes; the second one is what justifies the lint-rule investment.

**Going forward.**

- The contributor-framework promotion ([`docs/planning/contributor-framework.md`](../planning/contributor-framework.md)) should add an explicit dev-ready sub-task: a `verify.yml` step that reads every `@Controller('…')` decorator path in `backend/src/` and asserts it has a matching entry in both `frontend/vite.config.ts` proxy and `infra/modules/frontend_cdn/main.tf` ordered_cache_behavior. Drift fails the CI.
- When that lint rule lands, *both* Mid-flight 9 and Mid-flight 7 reference the rule as the canonical fix. The inline comments in vite.config.ts can be replaced with a one-line pointer.
- Any future contributor adding a new top-level NestJS controller path is the audience for the lint rule. The cost of the rule is small; the cost of each Mid-flight (debugging + decision-log entry + redeploy) is meaningful.

**What this isn't.** This is not a NestJS / Vite / CloudFront design problem. All three tools work as documented. The bug is the absent contract between them — three configuration surfaces that have to agree about which paths exist on the backend, with no shared source of truth today.

**Cross-reference.** P1 Mid-flight 9 (the original); P1 Mid-flight 11 (the `/resource*` vs `/resource/*` pattern within CloudFront); Mid-flight 6 above (same family — "two places agree by convention" footgun); [`docs/planning/contributor-framework.md`](../planning/contributor-framework.md) §4 (the planned lint rule).

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
