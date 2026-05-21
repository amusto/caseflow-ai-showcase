# Dev-Ready Task — P1.6 · Frontend login + gated page

**Phase:** [P1](../../../../ROADMAP.md#phase-1--multi-tenancy--real-auth-foundation-) · Sub-phase **P1.6**
**Decision log:** [`docs/decisions/phase-01-multi-tenancy-auth.md`](../../../../docs/decisions/phase-01-multi-tenancy-auth.md)
**Status:** dev-ready

## Goal

Wire the existing React+Vite+Redux+MUI frontend to the real auth
endpoints from P1.2–P1.5. End-to-end UX: register → see "awaiting
approval" gated page → admin approves → log in → land on the main app
shell. Implements Mid-flight 6 cookie switch atomically with the
frontend that consumes it. Google OAuth (P1.3) deferred — a placeholder
button reserves the spot in the login UI.

## Affected areas

**Backend (Mid-flight 6 cookie switch):**

- `backend/package.json` — add `cookie-parser` + `@types/cookie-parser`.
- `backend/src/main.ts` — `app.use(cookieParser())`.
- `backend/src/auth/auth.controller.ts` — `/auth/login` and `/auth/refresh` set `Set-Cookie: refreshToken=…; HttpOnly; Secure; SameSite=Strict; Path=/auth; Max-Age=604800`. `/auth/refresh` and `/auth/logout` read the cookie from `req.cookies?.refreshToken` (with body fallback for non-browser clients). Response body shape: `{ accessToken, tokenType, expiresIn }` (no `refreshToken` in body for browser clients per Pre-flight 13).
- `backend/src/auth/dto/refresh.dto.ts` — `refreshToken` becomes optional so the cookie path can work without a body.

**Frontend — new files:**

- `frontend/src/features/auth/types.ts` — `AuthUser`, `AuthState`, `LoginInput`, `RegisterInput`, decoded-JWT shape.
- `frontend/src/features/auth/jwt.ts` — `decodeAccessToken(token)` using `jwt-decode`.
- `frontend/src/features/auth/authApi.ts` — typed wrappers around `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`.
- `frontend/src/features/auth/authSlice.ts` — Redux slice (`accessToken`, `user`, `loading`, `error`) + thunks (`register`, `login`, `refresh`, `logout`).
- `frontend/src/features/auth/index.ts` — barrel.
- `frontend/src/pages/Login/LoginPage.tsx` + `index.ts`.
- `frontend/src/pages/Register/RegisterPage.tsx` + `index.ts`.
- `frontend/src/pages/AwaitingApproval/AwaitingApprovalPage.tsx` + `index.ts`.
- `frontend/src/routes/RequireAuth.tsx`, `RequireActiveAuth.tsx`, `PublicOnly.tsx` (per Pre-flight 14).

**Frontend — modified files:**

- `frontend/vite.config.ts` — proxy `/auth`, `/cases`, `/users` to `http://localhost:3000` (Pre-flight 13).
- `frontend/src/utils/axios-client/AxiosClient.ts` — replace mock-header interceptor with `Authorization: Bearer ${accessToken}` from Redux state; add response interceptor that attempts one refresh on 401, retries the original request once, dispatches `logout` if refresh also fails. `withCredentials: true` so cookies flow.
- `frontend/src/store/index.ts` — register `authReducer`.
- `frontend/src/App.tsx` — new routes + wrappers.
- `frontend/src/components/Layout/Layout.tsx` — show current user name + role + logout button.
- `frontend/src/pages/Home/Home.tsx` — minimal "you're in" placeholder (case list lands in a future phase).
- `frontend/src/utils/env.ts` — drop `DEV_MOCK_USER_ID`/`DEV_MOCK_USER_ROLE` (mock mode is backend-only now via `USE_MOCK_USER_GUARD` env flag).
- `frontend/.env.example` — same removal + a comment pointing at the backend flag.

## Acceptance criteria

- [ ] Vite dev server (`yarn dev:frontend`) proxies `/auth` and `/cases` to `localhost:3000` — `curl localhost:8080/auth/login -X POST ...` succeeds.
- [ ] Visiting `/login` and submitting valid creds for an ACTIVE user lands on `/` showing the main shell with the user's name and "ACTIVE" badge.
- [ ] Submitting valid creds for a PENDING_APPROVAL user lands on `/awaiting-approval`.
- [ ] Visiting `/` while unauthenticated redirects to `/login`.
- [ ] Visiting `/login` while authenticated as ACTIVE redirects to `/`.
- [ ] Visiting `/login` while authenticated as PENDING redirects to `/awaiting-approval`.
- [ ] Register form validates fields (email format, password length 8+, name 1–120 chars) and submits to `/auth/register` — on success, shows a success message and a link to `/login`.
- [ ] Logout button calls `/auth/logout`, clears Redux auth state, clears the `refreshToken` cookie (verifiable in DevTools → Application → Cookies), redirects to `/login`.
- [ ] Network tab on `/auth/login` shows `Set-Cookie: refreshToken=…; HttpOnly; SameSite=Strict; Path=/auth`. No `refreshToken` field in the JSON response body.
- [ ] On reload after login, the user is still ACTIVE — refresh cookie picked up automatically via the axios 401-retry interceptor on the first auth-required request.
- [ ] `yarn lint && yarn typecheck && yarn test` clean across the whole monorepo.

## Test plan

- **Manual smoke (the canonical P1 round-trip via UI):**
  1. Start backend (`yarn dev:backend`) and frontend (`yarn dev:frontend`).
  2. Browse to `http://localhost:8080/register`, fill the form, submit → expect success message.
  3. Browse to `/login`, log in → expect redirect to `/awaiting-approval`.
  4. In another terminal: `scripts/dev.sh db:promote your@email.com` (admin promotion).
  5. Hard-reload, navigate to `/login` (existing PENDING token gets a 401 on first request, gets refreshed-and-still-pending, redirects to /awaiting-approval — confirm the loop terminates cleanly).
  6. Log out, log back in → expect redirect to `/`. Layout shows your name + ACTIVE.
  7. Open DevTools → Application → Cookies → confirm `refreshToken` is HttpOnly + SameSite=Strict + Path=/auth.
- **Automated:** existing `auth.service.spec.ts` adjusted only if cookie behavior crosses into the service (it doesn't — controller handles all cookie I/O). Frontend component tests deferred.

## Out of scope (deferred)

- Google OAuth button wiring → **P1.3** (button placeholder ships in P1.6).
- Refresh token rotation across browser sessions / multiple tabs / etc. → already correct (server-side; rotation happens regardless of client).
- Case list / detail UI → starts in P2 (Customer model first).
- Form-level error message localization → defer.

## Resolved pre-flight decisions

- **Pre-flight 13** — Vite same-origin proxy for cookie posture.
- **Pre-flight 14** — Status-based routing (`PublicOnly`, `RequireAuth`, `RequireActiveAuth`).
- **Mid-flight 6** moves from "scheduled for P1.6" to "implemented in P1.6"
  for the refresh-token transport switch.

## How we'll work this sub-phase

Per Option 1: agent drives implementation, user reviews diffs.
