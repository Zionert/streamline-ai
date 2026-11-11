# Review: M1b — frontend authentication (feat/m1b-auth-frontend)

## Reviewer report — round 1
**Verdict:** CHANGES_REQUESTED
**Checks:** lint PASS, typecheck PASS, test PASS (shared 28, api 77, web 71)

### BLOCKER
- None.

### MAJOR
- `apps/web/src/features/auth/authApi.ts:31-39` + `apps/web/src/lib/api.ts:97-126` — **logout during an in-flight refresh does not log the user out** (client side and possibly server side). Sequence: access token expires -> a request 401s -> `/auth/refresh` (cookie C1) is in flight -> user clicks "Log out" -> `POST /auth/logout` goes out with C1 too, because the refresh response has not arrived yet. (a) Client: `logout()` runs `clearSession()` in `finally`, then the pending `performRefresh` resolves and calls `setSession(session)`, so the store goes back to `authenticated` with a valid 15-min access token, and `GuestRoute` on `/login` sends the user straight back into the app. (b) Server: refresh rotates C1 -> C2; logout then revokes C1 (already rotated, so nothing happens), and C2 stays valid. If the refresh response arrives after the logout's cookie-clearing response, the browser keeps C2, so the session survives on the server too. The window gets longer with the 400 ms rotated retry. "Log out actually logs out" is a security property (shared computers), and the fix is a few lines, so it should be fixed now and not deferred. Fix:
  1. Expose something like `waitForPendingRefresh(): Promise<void>` from `createApiClient` (returns `refreshInFlight?.then(() => undefined, () => undefined) ?? Promise.resolve()`), and `await` it in `logout()` before `POST /auth/logout`. Then logout sends the newest cookie (the server revokes C2), and any stale `setSession` happens before `clearSession`.
  2. Also add a session epoch/generation in `performRefresh`: capture it before the request, and call `setSession` only if it is unchanged (bump it in `clearSession`, or compare against a logout counter). This protects against any refresh that starts after `logout` began.
  3. Add a test: gate `/auth/refresh` with `deferred`, call `logout()`, resolve the refresh, then assert that `status === 'unauthenticated'` and that the logout request was recorded after the refresh request.

### MINOR
- `apps/web/src/features/auth/redirect.ts:25-31` — hardening: also reject pathnames starting with `/\` (browsers parse `/\host` like `//host`). The real risk is negligible: router state comes from `history.state`, which only same-origin code can set, and a cross-origin `pushState` would throw rather than navigate. Still, it is a cheap check plus one test row.
- `apps/web/src/lib/api.ts:18` — `REFRESH_TOKEN_ROTATED` (and `UNAUTHORIZED`) are string literals repeated in `apps/api/src/errors/app-error.ts` and in web. Move the error-code constants to `packages/shared` (shared contract) in a follow-up. Coder already flagged this; fine to defer.
- `apps/web/src/features/auth/LoginPage.tsx:31-32` / `RegisterPage.tsx:31-32` — after `setSession`, `GuestRoute` also renders `<Navigate>` to the same target, so the explicit `navigate()` runs as a duplicate replace navigation. It is harmless, but one of the two could go (for example, rely on `GuestRoute` alone, or keep the explicit call and accept it). No change required; just be aware of it.
- `apps/web/src/components/layout/AppLayout.tsx:15-19` — same pattern: `clearSession` makes `ProtectedRoute` redirect to `/login` with `from: '/'` before `navigate(LOGIN_PATH)` replaces it without state. Harmless.
- Build chunk 537 kB (no code splitting). Acceptable for now; consider lazy routes when board pages arrive.

### Verified / no issues found
- Single-flight: `refreshInFlight ??= performRefresh().finally(...)` is correct. Concurrent 401s share one request, the promise is cleared after settling, and the test checks one refresh and three replays.
- Rotated retry: one `sleep` and exactly one more `/auth/refresh`. A second failure (any 401) clears the session, and a non-401 rotated case is impossible. Tested for both the success and double-failure paths.
- `sentAccessToken` replay: replays without rotating when the store token changed while the request was in flight. A null token falls through to refresh, which is correct for guests and after logout. `authRetried` caps it at one replay per request, so there is no loop even if a newly issued token is byte-identical (same-second JWT).
- No refresh loops: `skipAuthRefresh` on the internal refresh call and `isAuthEndpoint` for login/register/logout/refresh, with regex edge cases tested (`/auth/refresh-tokens`, `/auth/me`, query string).
- Session is kept on network/5xx failures during a mid-session refresh and cleared on 401 or a schema mismatch. A single zod copy (4.6.7 everywhere) means `instanceof ZodError` is reliable.
- StrictMode: the `idle` -> `loading` switch is synchronous before the first await. A test with `<StrictMode>` checks exactly one refresh.
- Access token is only in Zustand memory, and a test checks that web storage and cookies are empty.
- Forms: `zodResolver` with shared schemas, `aria-invalid` + `aria-describedby`, `role="alert"` for server errors, submit disabled while pending, `from` preserved across the login/register link switch.
- Tests check behaviour (request counts, headers, store state, navigation targets), not just rendering. Injectable `sleep`/`adapter` keep them fast and deterministic.
- No `any`, no backend changes, `docs/` untouched, no privacy violations in tracked files.

### Done well
- Clean separation: transport/interceptor (`lib/api.ts`), API calls (`authApi.ts`), state (`stores/auth.ts`), routing guards.
- Precise reasoning in comments about why single-flight and the restore guard exist.
- The test helper `createMockAdapter` with ordered responder queues and `deferred` makes the concurrency tests easy to read.

## Reviewer report — round 2
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 28, api 77, web 77)

### BLOCKER
- None.

### MAJOR
- None. The round-1 logout-during-refresh item is fixed:
  - `logout()` now awaits `waitForPendingRefresh()` before `POST /auth/logout`. The server revokes the newest (rotated) cookie, and the refresh's `setSession` happens before `clearSession`. The test `features/auth/authApi.test.ts` checks the request order (refresh, then logout) and the final `unauthenticated` state.
  - `sessionEpoch` (bumped only in `clearSession`) is captured at the start of `performRefresh`. A stale success is discarded with `StaleRefreshError`, the interceptor rejects the original 401 and does not replay, and a stale failure no longer clears a newer session. All of this is tested.
  - `waitForPendingRefresh` never rejects, and logout still works when the pending refresh fails with 503 (tested).

### MINOR
- `apps/web/src/features/auth/authApi.ts:269-278` (residual, very unlikely): a refresh can still *start* during the logout POST, for example when another in-flight request gets a 401 while logout is posting. The epoch check makes the client discard its result. But on the server, that refresh may rotate C2 -> C3 before logout revokes C2. If its response arrives after the logout's cookie-clearing response, the browser keeps C3, and the next app start would restore the session. Optional hardening: set a "logging out" flag (or a module-level promise) before the POST that makes `refreshSession()` reject immediately until `clearSession` runs. Not blocking: it needs an expired access token plus a concurrent request within a ~50 ms window.
- `apps/web/src/stores/auth.ts:35` (theoretical): `login`/`register` do not bump `sessionEpoch`. A refresh that was in flight before a login could therefore overwrite (success) or clear (401) the new login session. In the current UI this cannot happen: guest pages make no API calls, and while restore is pending `GuestRoute` shows a loader instead of the form. Bumping the epoch in `setSession` would make it airtight. That is safe with single-flight, because `performRefresh` checks the epoch before calling `setSession`.
- `getRedirectTarget` now rejects `/\` paths, with a test row. Done.
- Deferred by agreement: shared error-code constants, code splitting, duplicate navigations.

### Done well
- The epoch guard combined with awaiting the pending refresh is a minimal, correct fix. Logout still never rejects and always clears the local session.
- The new tests check ordering and state with `deferred` gates instead of timing, so they are deterministic.
