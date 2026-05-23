# Dev-Ready Task — P2.2 · Admin notification on user registration

**Phase:** [P2](../../../../ROADMAP.md#phase-2--admin-section--customer-model-) · Sub-phase **P2.2**
**Decision log:** [`docs/decisions/phase-02-admin-and-personalization.md`](../../../../docs/decisions/phase-02-admin-and-personalization.md) — Pre-flights 5–8 (drafted alongside this spec; see "Resolved pre-flight decisions" below)
**Status:** dev-ready

## Goal

Stand up a backend `NotificationsModule` that emits three transactional emails, all backed by the same transport adapter:

1. **Admin registration alert** — every ADMIN user in the new registrant's tenant gets an email when `AuthService.register()` or the JIT path of `AuthService.googleSignIn()` creates a `PENDING_APPROVAL` user. CTA: deep-link to `/admin` (the P2.1 surface), where the admin can act in one click.
2. **Customer welcome** — the newly registered user gets an email confirming the account exists and is pending approval. Sets expectations: "an administrator will review your account; you'll get another email when it's approved."
3. **Customer approval-granted** — when `PATCH /users/:id/status` flips a user from `PENDING_APPROVAL` to `ACTIVE`, that user gets an email letting them know their account is live. CTA: deep-link to `/login`.

Transport is [Resend](https://resend.com) — selected over SES for setup cost (no DNS verification dance for the dev domain) and for the free tier comfortably covering portfolio-scale demo traffic. The `NotificationsService` is transport-agnostic: swapping to SES at P12 prod hardening is a single adapter swap, not a rewrite. A `LogTransport` fallback runs in environments without a Resend API key so local dev and CI never depend on a live external service.

P2.2 is backend-only. No frontend changes ship in this sub-phase — the P2.1 admin surface is already the email's CTA target. The new endpoints and entities surface entirely through `AuthService` + `UsersController.updateStatus`, both of which already exist.

## Why this architecture (the 90-second mental model)

Three layers, sharply scoped.

**Layer 1 — Transport** is the only piece that touches a network. `EmailTransport` is an interface (`send(message: EmailMessage): Promise<void>`). Two implementations ship: `ResendTransport` (the real path, used whenever `RESEND_API_KEY` is set) and `LogTransport` (the dev fallback that logs to stdout and resolves). Selection happens once at module load via a Nest factory provider — no runtime branching inside service methods.

**Layer 2 — Templates** are pure functions. Each template takes a typed input shape (`{ user, tenant, baseUrl }` or similar) and returns `{ subject, html, text }`. No template engine, no separate `.hbs` files, no I/O. Template literals + inline HTML are sufficient at this scale; if templates grow past ~50 lines or need shared partials, P2.followup graduates to a real templating library (recommendation: [`mjml`](https://mjml.io) for responsive HTML, with a precompile step). The unit tests assert exact subject lines and that the deep-link URL appears in both `html` and `text`.

**Layer 3 — `NotificationsService`** is the only thing other modules talk to. Three public methods, one per email type:

```ts
notifyOfNewRegistration({ user }: { user: User }): Promise<void>
notifyCustomerOfRegistration({ user }: { user: User }): Promise<void>
notifyCustomerOfApproval({ user }: { user: User }): Promise<void>
```

Each method looks up whatever it needs (the tenant's admin list, in the first case), renders the template, calls the transport. Errors are caught inside the service, logged with structured context, and swallowed — failure to send an email never propagates to the caller (Pre-flight 7).

**Call sites are three lines each:**

```ts
// AuthService.register, after createUser
void this.notifications.notifyOfNewRegistration({ user });
void this.notifications.notifyCustomerOfRegistration({ user });

// AuthService.googleSignIn, in the JIT-create branch
void this.notifications.notifyOfNewRegistration({ user });
void this.notifications.notifyCustomerOfRegistration({ user });

// UsersController.updateStatus, when previousStatus === PENDING && newStatus === ACTIVE
void this.notifications.notifyCustomerOfApproval({ user: updated });
```

The `void` is deliberate (Pre-flight 7) — the register response shouldn't wait on email transport latency, and email failure shouldn't break registration. `NotificationsService` self-contains all the error handling.

**What's deliberately NOT here.** No event-emitter pub/sub (over-engineering for three call sites). No outbox table with worker drain (real-world correct for at-least-once delivery, but heavyweight for a portfolio MVP — graduate at P11 when observability + retries land). No retries inside the transport (Resend's HTTP client doesn't retry by default; failures get logged and forgotten — the audit trail is the structured log, not a retry queue). No per-tenant email-template overrides (a customer-branding feature that lands as a stretch much later, if ever).

## Affected areas

### Backend — new files

**Module structure (`backend/src/notifications/`):**

- `notifications.module.ts` — registers the module. Imports `UsersModule` (for the tenant-admin lookup), `ConfigModule`. Provides `NotificationsService` + the `EmailTransport` factory. Exports `NotificationsService` so `AuthModule` and `UsersModule` can import it.
- `notifications.service.ts` — three public methods + private helpers. ~150 lines. All three methods follow the same shape: look up data → render template → call transport → catch + log on failure.
- `notifications.types.ts` — `EmailMessage` (`{ to: string[]; subject: string; html: string; text: string }`), input shapes for each template (`AdminRegistrationAlertInput`, `CustomerWelcomeInput`, `CustomerApprovalGrantedInput`), and the `EmailTransport` interface.

**Transport adapters (`backend/src/notifications/transport/`):**

- `email-transport.interface.ts` — re-exports `EmailTransport` for clarity.
- `resend.transport.ts` — wraps the Resend SDK (`resend` npm package). Constructor takes `apiKey` + `fromAddress`. Single `send(message)` method maps `EmailMessage` to the Resend payload.
- `log.transport.ts` — dev fallback. `send()` logs the full message at `info` level (subject, recipient list, first 200 chars of text) and resolves. Used when `RESEND_API_KEY` is unset.
- `transport.factory.ts` — Nest factory provider that returns `ResendTransport` when both `RESEND_API_KEY` and `RESEND_FROM_ADDRESS` are configured, otherwise `LogTransport`. Logs which transport was chosen at boot so the choice is visible in startup logs.

**Templates (`backend/src/notifications/templates/`):**

- `admin-registration-alert.template.ts` — exports `renderAdminRegistrationAlert(input)`. Subject: `"New user awaiting approval — [user name]"`. HTML includes user name, email, tenant name, registered-at timestamp, and a button-styled `<a>` linking to `${baseUrl}/admin`. Text version mirrors with the URL inline.
- `customer-welcome.template.ts` — exports `renderCustomerWelcome(input)`. Subject: `"Welcome to CaseFlow AI — your account is under review"`. HTML explains the approval flow ("an administrator at your organization will review your registration and approve access — you'll get another email once that's done"). No CTA button (nothing for the user to do yet).
- `customer-approval-granted.template.ts` — exports `renderCustomerApprovalGranted(input)`. Subject: `"Your CaseFlow AI account is approved"`. HTML confirms approval, includes a button-styled `<a>` to `${baseUrl}/login`.
- `templates.shared.ts` — small helpers: `escapeHtml`, `renderButton(url, label)`, `renderLayout(title, bodyHtml)`. Layout wraps each email body in a minimal responsive container with the CaseFlow AI wordmark and a footer ("Sent by CaseFlow AI — caseflow.musto.io"). Keeps each template focused on its actual content.

**Spec files:**

- `notifications.service.spec.ts` — covers all three methods (see Tests).
- `transport/log.transport.spec.ts` — verifies log format + resolves.
- `transport/resend.transport.spec.ts` — mocks the Resend SDK; verifies payload shape.
- `transport/transport.factory.spec.ts` — verifies factory choice based on env presence.
- `templates/admin-registration-alert.template.spec.ts` — subject + URL-in-body assertions.
- `templates/customer-welcome.template.spec.ts` — same shape.
- `templates/customer-approval-granted.template.spec.ts` — same shape.

### Backend — modified files

- `backend/src/app.module.ts` — register `NotificationsModule` alphabetically alongside the other feature modules.
- `backend/src/auth/auth.module.ts` — import `NotificationsModule` so `AuthService` can inject `NotificationsService`.
- `backend/src/auth/auth.service.ts` — inject `NotificationsService`. In `register()` (after the `createUser` line), fire `notifyOfNewRegistration` + `notifyCustomerOfRegistration` with `void` (non-blocking). In `googleSignIn()` JIT-create branch (after `createUserFromGoogle`), fire the same two calls.
- `backend/src/auth/auth.service.spec.ts` — extend with notification-fired assertions (mock `NotificationsService`, verify the two methods are called with the new user; verify register still resolves even when both methods reject).
- `backend/src/users/users.module.ts` — import `NotificationsModule` so `UsersController` can inject `NotificationsService` (controller-level injection is acceptable here; the controller is already the place that knows about the status transition).
- `backend/src/users/users.controller.ts` — inject `NotificationsService`. In `updateStatus`, capture the previous status before calling `usersService.updateStatus`. If previous was `PENDING_APPROVAL` and new is `ACTIVE`, fire `notifyCustomerOfApproval` with `void`. Other transitions (ACTIVE → ACTIVE, ACTIVE → DISABLED, PENDING → DISABLED) fire nothing.
- `backend/src/users/users.controller.spec.ts` — extend the PATCH-status tests to cover the new transition assertions.
- `backend/src/users/users.service.ts` — add `listAdminsByTenant(tenantId: string): Promise<User[]>` for the admin-recipient lookup. Returns `{ role: ADMIN, status: ACTIVE, tenantId }`. Excludes PENDING/DISABLED admins (they shouldn't be receiving operational mail).
- `backend/src/users/users.service.spec.ts` — extend with `listAdminsByTenant` coverage.

### Backend — package.json

- Add dependency: `resend@^4.0.0` (current major as of P2.2).
- No dev-dependency additions — the Resend SDK is mocked in unit tests via `jest.mock('resend', ...)`.

### Backend — environment

New env vars (documented in `backend/.env.example`):

- `RESEND_API_KEY` — Resend API key. Unset locally / in CI = `LogTransport` is selected (dev convenience). Set in deployed environments via Secrets Manager (P10 wiring already exists for this pattern; one new secret key joins the existing `JWT_SECRET` and `DATABASE_URL`).
- `RESEND_FROM_ADDRESS` — the `From:` header used on outbound emails. Format: `"CaseFlow AI <noreply@caseflow.musto.io>"`. Must match a verified Resend domain. Recommended value for dev: `noreply@caseflow.musto.io` (the existing dev domain). Recommended value for the demo onboarding ritual: same.
- `APP_BASE_URL` — root URL for deep-link construction. `https://caseflow.musto.io` in deployed dev; `http://localhost:5173` locally. Used by all three templates.

`backend/.env.example` gets a new block at the bottom commenting these three vars and pointing at `docs/decisions/phase-02-admin-and-personalization.md` Pre-flight 5 for context.

### Frontend

No frontend changes. The P2.1 `/admin` route and `PendingApprovalUsers` widget are already the email CTAs.

### Tests

**Backend unit tests (new):**

- `notifications.service.spec.ts`:
  - `notifyOfNewRegistration` looks up admins via `usersService.listAdminsByTenant`, calls transport once per admin (verified via mock-call shape), and resolves.
  - `notifyOfNewRegistration` resolves even when the tenant has zero admins (logs a warning; never throws). Defensive — shouldn't happen post-seed.
  - `notifyOfNewRegistration` resolves even when the transport rejects (caught + logged; doesn't propagate).
  - `notifyCustomerOfRegistration` calls transport once with the new user's email; resolves under all transport states.
  - `notifyCustomerOfApproval` calls transport once with the approved user's email; resolves under all transport states.
  - All three methods construct the correct `baseUrl` from `ConfigService.get('APP_BASE_URL')`.
- `transport/log.transport.spec.ts`:
  - Logs include subject + recipient list.
  - Resolves successfully.
- `transport/resend.transport.spec.ts`:
  - Maps `EmailMessage.to` (array) to Resend's `to` parameter.
  - Maps `from`, `subject`, `html`, `text` correctly.
  - Throws if the Resend SDK rejects (transport-layer error surfaces; service layer catches).
- `transport/transport.factory.spec.ts`:
  - Returns `LogTransport` when `RESEND_API_KEY` is unset.
  - Returns `LogTransport` when `RESEND_API_KEY` is set but `RESEND_FROM_ADDRESS` is unset (incomplete config = safer default).
  - Returns `ResendTransport` when both are set.
- Template specs (three files):
  - Subject matches the expected literal.
  - The `${baseUrl}/admin` (or `/login`) link appears in both `html` and `text`.
  - User name + email appear (where applicable).
  - No template injection — user-controlled fields (`user.name`, `tenant.name`) are HTML-escaped.

**Backend unit tests (extended):**

- `auth.service.spec.ts`:
  - `register` fires `notifyOfNewRegistration` + `notifyCustomerOfRegistration` with the persisted user.
  - `register` resolves with the user payload even if both notification methods reject (verified via promise rejection in the mock).
  - `googleSignIn` JIT branch fires both notifications; the existing-user branch does not.
- `users.controller.spec.ts`:
  - `updateStatus` from PENDING → ACTIVE fires `notifyCustomerOfApproval` exactly once.
  - `updateStatus` from PENDING → DISABLED fires nothing.
  - `updateStatus` from ACTIVE → DISABLED fires nothing.
  - `updateStatus` from ACTIVE → ACTIVE (no-op) fires nothing.
- `users.service.spec.ts`:
  - `listAdminsByTenant` returns only ADMIN + ACTIVE users in the given tenant.
  - Cross-tenant isolation: Tenant B's admins invisible to a Tenant A query.

**Manual smoke (end of Chunk B):**

1. `dev.sh demo:reset && dev.sh demo:seed` to start clean.
2. Register a new user via the frontend with a real email address.
3. Confirm both emails arrive: the admin email to the Tenant 1 admin (`sarah@abccorp.example` in seed data), the welcome email to the new registrant.
4. Approve the new user via `/admin`.
5. Confirm the approval email arrives at the new registrant.
6. Repeat once with `RESEND_API_KEY` unset locally; confirm the three log lines appear in backend stdout and no real email is sent.

## Acceptance criteria

- [ ] Registering a new user via `POST /auth/register` fires exactly one admin alert per ACTIVE admin in the new user's tenant and exactly one welcome email to the new user. Both fire independently and after `createUser` succeeds.
- [ ] Registering a new user via the Google JIT-create path (`POST /auth/google` with a never-seen-before email) fires the same two emails.
- [ ] Approving a `PENDING_APPROVAL` user to `ACTIVE` via `PATCH /users/:id/status` fires exactly one approval-granted email to that user.
- [ ] No other status transition fires an email (PENDING → DISABLED, ACTIVE → DISABLED, DISABLED → ACTIVE, etc.).
- [ ] `AuthService.register()` returns its `{ user: SafeUser }` payload synchronously without waiting on email transport. Transport latency does not delay the HTTP response.
- [ ] Email transport failure does not break registration or status updates. The user record is persisted, the HTTP response is `201` / `200`, and the failure surfaces only in backend logs.
- [ ] Cross-tenant isolation: a registration in Tenant A does not email Tenant B's admins.
- [ ] In environments without `RESEND_API_KEY`, `LogTransport` is selected at boot and every "send" is a structured log line instead of a network call. No test or local dev session requires a live Resend account.
- [ ] All three email templates render the recipient name (where applicable), the action URL, and a CaseFlow AI footer in both HTML and text. User-controlled fields are HTML-escaped — no `<script>` payload in a name field can render as live HTML in the recipient's mail client.
- [ ] `yarn workspace @caseflow-ai/backend test` clean.
- [ ] `yarn workspace @caseflow-ai/backend lint` clean.
- [ ] No proxy / CloudFront behavior changes (no new HTTP routes).
- [ ] No frontend changes.

## Resolved pre-flight decisions

The four pre-flights below should be drafted into `docs/decisions/phase-02-admin-and-personalization.md` alongside this spec and committed in the same change.

**Pre-flight 5 (P2) — Email transport: Resend now, SES at P12 prod hardening.**

- *Decision.* `ResendTransport` is the day-one transport. SDK: `resend@^4.0.0`. From-address must match a verified Resend domain (`caseflow.musto.io` already verified for the project's dev domain — one-time DNS setup completed during P2.2 implementation, not in this spec). Free tier (3,000 emails/month, 100/day) covers portfolio demo traffic with order-of-magnitude headroom.
- *Alternative rejected — SES from day one.* Production-grade and IAM-native, but the setup tax is ~14hr including DNS verification, suppression list config, sandbox-out-of-sandbox application, and bounce-handler wiring. None of that work is portfolio-defensible for an MVP; all of it lands defensibly at P12 when prod hardening is the actual goal. Hard rule: SES is the prod transport, Resend stays the dev transport, the swap happens once at P12.
- *Alternative rejected — both adapters ship now, env-selected.* Builds twice the code for half the value. The current adapter interface (`EmailTransport.send`) already supports the future SES adapter as a drop-in; nothing about Resend-first prevents it.
- *Revisit if.* Resend reliability drops below acceptable for the demo audience (>1% delivery failures observed in dev logs), or the project crosses Resend's free-tier limits — either case triggers the early SES swap.

**Pre-flight 6 (P2) — All three transactional email types ship together in P2.2.**

- *Decision.* P2.2 ships three email types in one pass: admin registration alert (the original P2.2 scope), customer welcome on registration (new in this spec), customer approval-granted on PATCH-status transition (new in this spec).
- *Why not defer welcome + approval to a post-P2.2 evaluation.* The original ROADMAP open call was "decide based on demand from the LinkedIn audience." Evaluating that demand requires the register-and-approve flow to be conversational — the admin alert alone leaves the registrant in silence between submitting the form and getting approved, which is the exact moment of highest drop-off risk for the demo. Three emails together close the loop. Ship cost is ~2x admin-alone (template work doubles, integration work is identical), and shipping them together avoids a follow-up sub-phase that would otherwise sit on the backlog through three later phases.
- *Alternative rejected — admin alert only, customer emails deferred.* Tempting for speed, but creates a silent-system gap right where the demo's narrative ("we approve responsibly") wants to be loudest. The cost saved is real but the cost incurred (a half-finished story for demo viewers) is worse.
- *Revisit if.* Resend free-tier limits become a real constraint — at that point, dropping the welcome email is the lowest-value drop. Approval-granted is the most-valuable retain.

**Pre-flight 7 (P2) — Direct injection + non-blocking dispatch; no event bus, no outbox.**

- *Decision.* `NotificationsService` is injected directly into the call sites (`AuthService`, `UsersController`). Each call site uses `void` to fire without awaiting. All error handling lives inside `NotificationsService` — methods catch, log with structured context (user id, notification type, transport error), and resolve successfully.
- *Alternative rejected — `@nestjs/event-emitter` pub/sub.* In-process events sound nice ("decouple AuthService from NotificationsService") but the decoupling claim is theoretical — both modules ship in the same deployable artifact and three call sites is too few to amortize the indirection cost. The "easier to add more listeners later" argument is real but premature: P7 will not extend P2.2's emitter; it will build a different mechanism (outbox → SQS).
- *Alternative rejected — outbox pattern (DB row written in same TX as user-create, worker drains and sends).* Real-world correct for at-least-once delivery, but P2.2 is portfolio-MVP. The cost (a new table, a worker process, retry semantics, observability) does not earn the value (at-least-once guarantees) at this stage. P11 introduces observability + the operational reporting that would make outbox visible; P12 introduces prod hardening that would make at-least-once material. P2.2's "fire and forget + log on failure" is defensible for the actual user volume.
- *Revisit if.* The decision log accumulates evidence that emails are silently failing in production (deliverability reports, support requests). At that point, the outbox migration is well-scoped: `NotificationsService` stays as the only API surface; internally it changes from direct-call to enqueue.

**Pre-flight 8 (P2) — Admin recipients: lookup, not env-var.**

- *Decision.* `notifyOfNewRegistration` resolves recipients dynamically by calling `usersService.listAdminsByTenant(user.tenantId)`. Every ACTIVE admin in the new registrant's tenant gets a copy. No `ADMIN_NOTIFY_EMAIL` env var.
- *Why not env-var.* The ROADMAP entry called for `ADMIN_NOTIFY_EMAIL`, but env-vars for recipient routing become wrong the instant a second tenant or a second admin exists. Both already exist in the seed data. Dynamic lookup is one extra query per registration (acceptable cost; cached at the service level if it ever becomes hot) and gets multi-tenancy right by default.
- *Alternative rejected — single global `ADMIN_NOTIFY_EMAIL` env-var.* Works for the single-admin demo profile, breaks the moment a second tenant signs up via the LinkedIn invitation. The fix would arrive as a Mid-flight, which is exactly the kind of architectural drift this decision log exists to prevent.
- *Alternative rejected — per-tenant config table (`tenant_notification_settings`).* Right answer eventually (mute/unmute per admin, digest mode, etc.), but premature in P2.2. Add when a real preference emerges.
- *Revisit if.* The admin list grows past ~5 per tenant — at that point, "all admins" becomes noisy and a `notification_preferences`-shaped table earns its keep.

## Out of scope (deferred)

- **Bounce + complaint handling.** Resend reports bounces via webhook, but we don't ingest them in P2.2. A bounced email surfaces only in the Resend dashboard. P11 (observability) is the natural home for webhook ingestion + bounce-row inserts.
- **Email open / click tracking.** Resend supports it via header opt-in; we leave it off. Adding tracking has implications for the marketing-vs-transactional distinction (and downstream for the SOC II readiness work in `docs/planning/soc2-readiness.md`); deciding it deserves its own pre-flight, not a P2.2 implementation detail.
- **Per-tenant email templates / branding.** Customer-branded emails ("Sent by Acme Support, powered by CaseFlow AI") is a paid-feature shape. Deferred indefinitely; revisit only if a tenant explicitly asks.
- **Retries + dead-letter queue.** Single-shot transport call; failure logs and stops. The audit trail is the log, not a retry queue. The outbox + retries upgrade lands when at-least-once delivery is a real requirement (post-P11).
- **Digest mode.** "Email me a daily summary of pending registrations" instead of per-event. Add when the live demo's pending queue regularly exceeds ~10/day; until then, per-event is the right granularity.
- **In-app notifications.** A bell icon in the app bar with a notification dropdown is a separate surface from email and lands as its own phase (currently not on the roadmap; would slot near P11). P2.2 stays email-only.
- **Localized templates.** All three templates are English-only. i18n is a P14 (customer-app) concern at earliest.
- **Disable-account email.** No email on PENDING → DISABLED or ACTIVE → DISABLED transitions. The product surface for "your account was disabled" lives in P12 prod hardening alongside the rest of the support-flow polish.
- **SES adapter implementation.** The interface is in place; the SES adapter file lands in P12.
- **Resend webhook signature verification.** Not needed in P2.2 because we don't ingest any Resend webhooks. Lands with bounce handling.

## How we'll work this sub-phase

Three chunks with review checkpoints between each. Same pattern as P1.7, P1.8, P2.1.

### Chunk A — Notifications module + transports + templates (no integration yet)

1. `notifications.module.ts`, `notifications.service.ts`, `notifications.types.ts` — scaffolded with empty bodies / stubbed methods.
2. `transport/email-transport.interface.ts`, `transport/log.transport.ts`, `transport/resend.transport.ts`, `transport/transport.factory.ts`.
3. `templates/*.template.ts` (three) + `templates.shared.ts`.
4. Specs for all of the above. Transport specs mock the Resend SDK; template specs assert subject + URL-in-body.
5. Register `NotificationsModule` in `app.module.ts`. Boot the backend locally; confirm the startup log line announces which transport was selected.
6. Add `resend` to `backend/package.json`; run `yarn install`.

**Review checkpoint 1.** Module loads. Transport factory chooses correctly under each env state. Templates produce the expected subject lines and bodies. No call site wired yet — pure infrastructure.

### Chunk B — Wire the three call sites + end-to-end smoke

1. `AuthModule` imports `NotificationsModule`; `AuthService` injects `NotificationsService`; `register()` + `googleSignIn()` JIT branch fire the two new-user notifications with `void`.
2. `UsersModule` imports `NotificationsModule`; `UsersController` injects `NotificationsService`; `updateStatus` captures previousStatus, fires `notifyCustomerOfApproval` on the PENDING → ACTIVE transition only.
3. `users.service.ts` gains `listAdminsByTenant`; spec extends.
4. `auth.service.spec.ts` + `users.controller.spec.ts` extend with the notification-fired assertions and the negative-transition assertions.
5. Manual smoke: real registration through the frontend with a real email address; verify all three email types deliver. Then with `RESEND_API_KEY` unset locally: verify the three log lines instead.
6. Manual smoke for the JIT-create path: Google sign-in with a never-seen-before account; verify the same two emails fire.

**Review checkpoint 2.** Three email types delivering to a real inbox under the deployed dev environment. Failure cases (transport down, no admins in tenant) surface only in logs. Registration response time visibly unchanged from pre-P2.2 (transport latency does not block).

### Chunk C — Documentation + decision-log pre-flights + ROADMAP flip

1. Draft Pre-flights 5–8 in `docs/decisions/phase-02-admin-and-personalization.md` (bodies stubbed in this spec).
2. `backend/.env.example` updated with the three new env vars + a comment block linking to Pre-flight 5.
3. ROADMAP `Current state` block: flip "Last shipped" to mention P2.2; advance the By-phase row to `██░░░░░░░░ 29% 2/7`; flip the P2.2 sub-phase bullet from `⬜` to `✅` with the implementation summary.
4. `docs/planning/soc2-readiness.md` — add a one-line entry under "Compliance-touched surfaces" noting that the new audit-log surface (template renders + transport calls) lands in P11; P2.2's logging is informal.

**Review checkpoint 3.** Pre-flights captured. ROADMAP flipped. `.env.example` updated.

## Risks and how we'll handle them

- **Resend domain not verified at implementation time.** The from-address must match a verified Resend domain or all sends fail at the API. Mitigation: domain verification is a one-time DNS task that happens before Chunk B's manual smoke. If verification is incomplete, the manual smoke uses a Resend test address (`onboarding@resend.dev`) so the wiring is still verified; the production from-address swap happens once the DNS propagates.
- **Email arrives in spam.** First-day deliverability is unpredictable. Mitigation: SPF + DKIM are set up during domain verification (both required by Resend's verification flow, so this is enforced by their setup wizard, not by our discipline). DMARC stays at `p=none` for P2.2 — strict DMARC lands at P12 with the rest of the prod hardening. If spam classification is observed during the manual smoke, log a Mid-flight and continue; deliverability tuning is its own multi-week concern that doesn't gate P2.2.
- **Template HTML rendering varies across mail clients.** Outlook, Gmail web, iOS Mail, and Gmail mobile each interpret email HTML/CSS slightly differently. Mitigation: keep the templates simple (no CSS Grid, no flexbox, no JS, no remote stylesheets — inline `style=""` only, table-based layout if alignment matters). The `templates.shared.ts` layout helper enforces this by being the only HTML structure entry point. Cross-client visual QA is a manual one-time pass in Chunk B against Gmail + Apple Mail; if a third client misrenders meaningfully, that's a follow-up Mid-flight.
- **A registration burst exhausts Resend's daily free-tier (100 emails / day).** Mitigation: hard rate-limit at the service layer is not in P2.2 scope. The free-tier ceiling is high enough for portfolio traffic that exceeding it would itself be a strong signal — log a Mid-flight, upgrade Resend, continue. If the LinkedIn invitation post (#62) drives a spike, this is the failure mode to watch.
- **HTML injection via `user.name`.** A registrant could submit `<script>alert(1)</script>` as their name. Mitigation: `escapeHtml` in `templates.shared.ts` is mandatory for every user-controlled field that enters the HTML body. Template specs explicitly test escaping. The text version of each template renders names raw (no HTML context to inject into).
- **Approval-granted email fires on the wrong transition.** Distinguishing PENDING → ACTIVE from ACTIVE → ACTIVE requires capturing previous status before the save. Mitigation: explicit `users.controller.spec.ts` coverage of all four transition cells (PENDING → ACTIVE, PENDING → DISABLED, ACTIVE → DISABLED, ACTIVE → ACTIVE). Negative tests assert no-call as strongly as positive tests assert call.
- **Notification methods leak to logs in tests.** When a unit test triggers `register`, the real `NotificationsService` would attempt to send via `LogTransport` (which logs). Mitigation: mock `NotificationsService` entirely in `auth.service.spec.ts` and `users.controller.spec.ts` — never let the real service run inside a unit test. `notifications.service.spec.ts` is the only place the real service runs, and it mocks the transport instead.
- **NotificationsModule + UsersModule circular import.** `NotificationsService` depends on `UsersService` (for `listAdminsByTenant`); `UsersModule` imports `NotificationsModule` (for the controller's approval-granted call). Mitigation: this is a real circular dependency if both modules import each other's modules. Resolve by using `forwardRef(() => UsersModule)` in `NotificationsModule.imports` and `forwardRef(() => NotificationsModule)` in `UsersModule.imports`. Document the `forwardRef` pair inline; both are well-understood Nest patterns.

## Followup

After P2.2 ships:

- **#67 (Evaluate customer-facing email needs)** auto-resolves — the three email types already ship together in P2.2.
- **P2.3 (CustomerOrganization + CustomerContact entities)** is the next sub-phase. P2.2 establishes nothing P2.3 depends on; the two work in parallel sequence rather than serial dependence.
- **Resend bounce webhook ingestion** is a P11 followup (observability theme). Open question: do we persist bounce events to a `notification_log` table or just to CloudWatch? P11 decides.
- **SES adapter** lands at P12 prod hardening. The interface is in place; the adapter file is the only thing missing.
- **Article candidate.** "From event-emitter to outbox: when a portfolio MVP graduates to at-least-once delivery" is a defensible follow-up article. Currently fits the "Decision logs as engineering artifacts" series shape (Article 02 draft is #76, currently pending).
- **ROADMAP P2 status flips to `██░░░░░░░░ 29% 2/7` after P2.2.**

## Confidence

**Confidence:** 95%. Risks are well-bounded (the circular-import gotcha is the one with real teeth; everything else is logging or template polish). The architecture matches the existing codebase patterns — direct service injection, ConfigService-mediated env, structured test coverage. The transport-layer abstraction is small enough to be unambiguous. The four pre-flight decisions answer every open architectural call that touched P2.2; nothing about the implementation requires "judgment calls" beyond what's captured here.
