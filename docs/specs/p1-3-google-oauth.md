# Dev-Ready Task — P1.3 · Google OAuth + JIT provisioning

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.3**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Add "Sign in with Google" alongside the existing email + password flow.
New Google users get JIT-provisioned into the default tenant with
`role=CUSTOMER`, `status=PENDING_APPROVAL` — the same status gate that
password registrations land in. Existing email + password users who
sign in with the same Google email get **linked** to their existing
account, not a duplicate.

P1.3 closes the last open P1 sub-phase. After this, P1 is 6/6.

## Affected areas

**Backend — new files:**

- `backend/src/auth/dto/google-sign-in.dto.ts` — single field `idToken` (string, length-bounded).
- `backend/src/auth/google-verifier.ts` — wraps `google-auth-library`'s `OAuth2Client.verifyIdToken({idToken, audience: GOOGLE_CLIENT_ID})`. Returns `{ email, name, sub, emailVerified, picture }` or throws.

**Backend — modified files:**

- `backend/src/auth/auth.service.ts` — new `googleSignIn(idToken)` method:
  1. `verifier.verify(idToken)` → returns claims, throws on invalid.
  2. If `!emailVerified` → throw 401 `EMAIL_NOT_VERIFIED`.
  3. Look up user by email. If found, link (Pre-flight 20) by stamping `googleSub` if not already set; issue token pair.
  4. If not found, JIT-create user via `usersService.createUserFromGoogle({ email, name, googleSub, picture? })`. Defaults: `role=CUSTOMER`, `status=PENDING_APPROVAL`, attached to default tenant (Pre-flight 9).
  5. Issue token pair (access JWT + refresh token cookie) via the existing `issueTokenPair` path. Same status claim handling — PENDING users get tokens, DISABLED don't.
- `backend/src/auth/auth.controller.ts` — new `@Public() @Post('google') @HttpCode(200) googleSignIn(@Body() dto, @Res passthrough)`. Mirrors the cookie-issuing shape of `/auth/login`.
- `backend/src/users/users.service.ts` — new `createUserFromGoogle()` that takes Google claims and creates a user with `passwordHash` set to a random opaque hash that cannot match any real password (the user can never log in with password until they explicitly set one — future work). Method also handles tenant attach.
- `backend/src/users/entities/user.entity.ts` — new optional column `googleSub: string | null` (unique-indexed when not null). Used for linking and future "is this account Google-bound" checks.
- `backend/package.json` — add `google-auth-library` to dependencies.

**Frontend — new files:**

- `frontend/src/features/auth/authApi.ts` — add `googleSignIn(idToken: string)`.
- `frontend/src/features/auth/authSlice.ts` — add `googleSignInThunk` mirroring `loginThunk`.

**Frontend — modified files:**

- `frontend/src/pages/Login/LoginPage.tsx` — replace the disabled "Coming in P1.3" button with a real `<GoogleLogin>` from `@react-oauth/google`. On success → dispatch `googleSignInThunk` with the credential.
- `frontend/src/main.tsx` — wrap the app in `<GoogleOAuthProvider clientId={env.GOOGLE_CLIENT_ID}>`. Only when `GOOGLE_CLIENT_ID` is set — otherwise omit the provider so the app still renders if the client ID is missing in dev.
- `frontend/package.json` — add `@react-oauth/google` to dependencies.
- `frontend/.env.example` — `VITE_GOOGLE_CLIENT_ID` already present (placeholder); add an inline comment pointing at the bootstrap instructions.

**Env (deploy):**

- `backend/.env(.example)` — add `GOOGLE_CLIENT_ID=` (empty placeholder; bootstrap instructions in spec below).
- `infra/live/dev/backend/terragrunt.hcl` — add `GOOGLE_CLIENT_ID` to `environment_variables`. **It's a public-shape identifier** (not a secret) — fine in env vars, no Secrets Manager needed.

## Bootstrap — Google Cloud Console (one-time)

1. https://console.cloud.google.com → pick or create a project (e.g. "caseflow-ai-dev").
2. **APIs & Services → OAuth consent screen.** Configure as "External" (any Google account can sign in), scopes: `email`, `profile`, `openid`. Add yourself as a test user while in "Testing" status.
3. **APIs & Services → Credentials → Create OAuth client ID.**
   - Application type: Web application.
   - Authorized JavaScript origins:
     - `http://localhost:8080` (frontend dev)
     - `https://caseflow.musto.io` (deployed frontend)
   - Authorized redirect URIs: leave blank — we use the JS SDK's pop-up / one-tap flow, not redirect-based auth.
4. Copy the **Client ID** (looks like `123456-abcdef.apps.googleusercontent.com`).
5. Set in both env files:
   - `backend/.env`: `GOOGLE_CLIENT_ID=123456-abcdef.apps.googleusercontent.com`
   - `frontend/.env`: `VITE_GOOGLE_CLIENT_ID=123456-abcdef.apps.googleusercontent.com`
6. After deploy, also update `infra/live/dev/backend/terragrunt.hcl`'s `environment_variables` and re-apply.

## Acceptance criteria

- [ ] First-time Google sign-in with a fresh Google account creates a `User` row with `googleSub` populated, `passwordHash` opaque-random, `status=PENDING_APPROVAL`, attached to default tenant.
- [ ] Subsequent Google sign-ins with the same account return tokens without creating a new user.
- [ ] Google sign-in with an email that matches an existing password-registered user **links** by stamping `googleSub` on the existing row; same user, both methods now work.
- [ ] Google sign-in with an unverified Google email returns `401 EMAIL_NOT_VERIFIED`.
- [ ] Google ID token with wrong audience (different `GOOGLE_CLIENT_ID`) returns `401 INVALID_TOKEN`.
- [ ] Frontend login page renders a real Google button when `VITE_GOOGLE_CLIENT_ID` is set, falls back to the disabled placeholder otherwise.
- [ ] Successful Google sign-in flows through the same `loginThunk → authSlice` pipeline; user lands on `/awaiting-approval` (or `/` if status is ACTIVE).
- [ ] Tests cover: verify success, verify failure (bad audience, expired, unverified email), JIT create, link to existing, idempotent re-sign-in.
- [ ] `yarn workspace @caseflow-ai/backend lint && typecheck && test` clean.
- [ ] `yarn workspace @caseflow-ai/frontend lint && typecheck && test` clean.

## Resolved pre-flight decisions (see decision log)

- **Pre-flight 20 — Email linking on Google sign-in.** Same email → same user (stamp `googleSub` on existing row). Rejected the "create separate Google user" path (would violate email-global-unique invariant from Pre-flight 7) and the "block Google sign-in if email exists" path (worse UX).
- **Pre-flight 21 — Google users land PENDING_APPROVAL.** Same gate as password registration (Pre-flight 8). A bad actor with a free Google account shouldn't auto-bypass admin approval.
- **Pre-flight 22 — Token verification via `google-auth-library`.** Battle-tested, handles JWKS rotation + signature + audience + issuer + expiry in one call. Manual JWKS verification rejected as reinventing.

## Out of scope (deferred)

- Allowing a Google-linked user to also set a password later — UX consideration for future work.
- Account unlinking (disconnect Google from a linked user). Future hardening.
- Google Workspace domain restriction (`hd` claim filter). Useful for B2B; future SaaS-tenancy work.
- "Sign up with Google creates a new tenant" flow. Pre-flight 9 says everyone lands in the default tenant for now.

## How we'll work this sub-phase

Per Option 1: agent drives implementation, user reviews diffs.
