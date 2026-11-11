# Review: M1a — backend authentication (`feat/m1a-auth-backend`)

## Reviewer report — round 1
**Verdict:** CHANGES_REQUESTED
**Checks:** lint PASS, typecheck PASS, test PASS (shared 27, api 71, web 5), format:check PASS, build PASS

Dev DB safety verified by hand: put a marker row into `streamline.User` before `pnpm test`. It was still there afterwards and `RefreshToken` stayed empty. Tests ran against `streamline_test`. Also checked that `process.loadEnvFile('.env')` in `prisma.config.ts` does NOT override the `DATABASE_URL` that global setup injects, so `migrate deploy` hits the test DB. Marker row removed after the check.

### BLOCKER
- none

### MAJOR
1. `apps/api/src/services/auth.service.ts:103-124` + `apps/api/src/controllers/auth.controller.ts:72-79` — **a harmless concurrent refresh logs the user out on every device.** Any second use of a token that was just rotated goes to `handleTokenReuse`, whether it comes through the `revokedAt` branch or the `rotated === null` race branch. This happens in normal use, not only under attack. Examples: the browser restores several tabs at once, or two tabs reload at the same time. Each tab has its access token only in memory, so each calls `/auth/refresh` on startup with the same cookie. Single-flight inside one tab (the proposed frontend fix) does not cover several tabs. On top of that, the losing response calls `clearCookie`, which can delete the cookie the winning response has just set.
   → Fix: add a short grace window (e.g. `REFRESH_REUSE_GRACE_SECONDS`, default about 30, validated in `env.ts`). A token that was revoked *by rotation* (`replacedById != null`) less than that long ago is treated as a benign race. Return 401 (or a distinct code such as `REFRESH_TOKEN_ROTATED`), do NOT revoke the family, and do NOT clear the cookie. Do the same in the `rotated === null` branch. Keep full family revocation for older reuse and for tokens revoked by logout or by a previous family revocation.
   → Add an integration test that sends two refreshes with the same cookie via `Promise.all`. Expect one 200, one 401, and the winner's new token still usable. Add a second test where the grace period has passed: move `revokedAt` back in the DB and expect all sessions revoked.
   → If the team decides to keep the strict behaviour from the brief, that has to be an explicit decision recorded in `docs/DECISIONS.md`. M1b would then need cross-tab coordination (e.g. `navigator.locks`). In that case only the `clearCookie` issue remains to fix.

### MINOR (optional)
- `apps/api/src/services/password.service.ts:28` — the dummy hash is created lazily, so the first unknown-email login after startup does hash + compare (about 2x slower). That reveals one bit, once per process. Start the promise eagerly in `createPasswordHasher`.
- `packages/shared/src/schemas/auth.ts:46` — login accepts up to 256 chars, and bcryptjs silently truncates to 72 bytes (checked: comparing 200 x `a` against a hash of 72 x `a` returns `true`). The impact is negligible because registration caps passwords at 72 bytes. For consistency, reject over 72 bytes at login the same way, or just leave a comment.
- `apps/api/src/middleware/rate-limit.ts` — no `trust proxy` setting. Behind a reverse proxy, every client would share one IP, and 10 failed logins by anyone would block everyone. Already listed as known debt. Add a `TRUST_PROXY` env option before deployment.
- `apps/api/src/services/auth.service.ts:78` — register returns 409 for an existing email, so it reveals which emails have accounts. This is an accepted UX trade-off, and it is rate-limited and timing-equalised because the hash is computed first. Note it in the thesis notes.
- `apps/api/src/services/token.service.ts:41` — consider adding `iss`/`aud` claims, and checking them, before more token types appear (e.g. socket auth).
- `apps/api/src/repositories/refresh-token.repository.ts:37` — no DB-level test of `rotate`'s conditional update under concurrency. The service test only mocks `null`. The `Promise.all` test from MAJOR #1 would cover it.
- Old refresh tokens are never cleaned up (known debt). Expired/revoked rows will grow without limit; add a periodic cleanup later.

### Done well
- Token handling is correct: 256-bit random opaque refresh token, only its SHA-256 stored, HS256 with an algorithm allowlist (the `alg: none` test is there), `sub` and `exp` required, secret of at least 32 chars validated at startup.
- Atomic rotation via conditional `updateMany ... revokedAt: null` inside a transaction. This is safe under READ COMMITTED: the second updater blocks, re-checks the condition and gets `count = 0`.
- Enumeration and timing on login: dummy bcrypt compare, identical 401 bodies (tested), no Set-Cookie on failure.
- Cookie: HttpOnly, SameSite=Strict, Secure in production, path matches the Vite `/api` rewrite, `Cache-Control: no-store`. The access token is never in a cookie (tested).
- Clean layering: Prisma is used only in repositories (plus the test-only truncate helper), manual DI in `createApp`, services unit-testable with fakes.
- Errors are handled the same way everywhere: 401/409/429 go through `AppError`, and the rate limiter calls `next(err)`, so all responses use the existing error format.
- Test DB safety: requires `_test` suffix, never reads `process.env.DATABASE_URL`, `fileParallelism: false`, truncate before each test, `src/test` excluded from the build. CI uses `DATABASE_URL_TEST` against a Postgres service whose DB name matches, and `prisma generate` still works without `DATABASE_URL`.
- Logger redaction of the auth header, cookies and Set-Cookie, with a test.
- Thorough integration tests on behaviour: rotation, reuse revoking the whole family, isolation from other users, expiry, logout idempotency, cookie attributes in prod and non-prod, both rate limiters.
- No `any`. No AI traces in tracked or untracked repo files. `CLAUDE.md` and `.claude/` are excluded only via `.git/info/exclude`. No changes to frontend or `docs/`.

### Required fixes (numbered)
1. Grace window for just-rotated tokens: no family revocation and no cookie clearing during the window. Add a concurrent-refresh integration test and a test for reuse after the window (MAJOR #1). Alternative: the user explicitly accepts the strict behaviour, the decision is recorded, and only the cookie clearing is fixed.

---

## Reviewer report — round 2
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 28, api 77, web 5), format:check PASS. The api suite was run 4 times in a row with 77/77 every time, no flakiness.

Dev DB safety re-verified: a marker row put into `streamline.User` survived all 4 test runs and `RefreshToken` stayed at 0. Marker removed afterwards. No AI traces in changed or new files. No changes under `docs/` or `apps/web`.

### Round-1 required fix — resolved
- MAJOR #1 (concurrent refresh logs the user out on every device) is fixed in `apps/api/src/services/auth.service.ts:73-142` and `apps/api/src/controllers/auth.controller.ts:73-85`.
  - **What counts as benign:** the token was revoked *by rotation* (`replacedById !== null`) less than `REFRESH_REUSE_GRACE_SECONDS` ago, or the request lost the conditional update. Either case returns `REFRESH_TOKEN_ROTATED` (401) with no mass revoke and the cookie kept.
  - **Still treated as theft:** tokens revoked by logout or by a mass revoke (`replacedById` null), reuse after the window, and any reuse when grace = 0.
  - **Config:** `env.ts` validates the window as 0–300, default 30.
  - **Tests:** a real-DB `Promise.all` test (exactly one 200 and one ROTATED with no Set-Cookie; the winner's token and the other device keep working; only one row revoked), reuse after the window, grace = 0, logout-token reuse inside the window, and unit tests for every branch.
- Minors applied: the dummy hash is created up front; login rejects passwords over 72 bytes (same check as register).

### Security analysis of the grace window
- **Attacker uses a stolen, already-rotated token within the window:** gets ROTATED and **no new token**. Nothing is issued and nothing leaks. Detection is only postponed: any use after the window revokes everything.
- **Attacker rotates first, then the legitimate client presents the old token within the window:** gets ROTATED instead of revoking everything. Theft detection now depends on the client presenting the old token again *after* the window, or on the next login. This is the known, accepted cost of a reuse interval (the same trade-off as Auth0's "reuse interval"). The window is short and configurable, and grace = 0 restores the strict behaviour. Acceptable; it does not block approval.
- **Lost race where the competing revocation was a logout or mass revoke rather than a rotation:** ROTATED is returned, but no token is issued, so nothing bad happens.
- **Leakage:** the distinct `REFRESH_TOKEN_ROTATED` code only tells the caller that a token they already hold was recently rotated. No user data, no timing difference worth mentioning.

### BLOCKER
- none

### MAJOR
- none

### MINOR (optional / for later milestones)
- Frontend contract for M1b (for the documenter): on `REFRESH_TOKEN_ROTATED`, wait briefly and retry **once**. In the multi-tab case the browser now holds the winner's new cookie, so the retry succeeds. If the retry fails again, treat the user as logged out and do not keep retrying. Repeated retries after the window would trigger a mass revoke on the user's own sessions. Recommend recording the grace window in `docs/DECISIONS.md` as the rationale for the trade-off.
- `isRecentlyRotated` compares `revokedAt` (written by the app clock) with `Date.now()`. That is fine for one instance; if the API is ever scaled out, clock skew between instances slightly changes the effective window. Optional: use the DB clock.
- Accepted known debt is unchanged: trust proxy, `iss`/`aud`, token cleanup, 409 on register, in-memory rate-limit store.

### Done well
- The fix is minimal and precise. The benign-vs-theft classification relies on `replacedById`, which round 1 already stored, so no schema change was needed.
- The controller keeps the cookie only for ROTATED; every other refresh failure still clears it.
- The concurrency test runs against the real database and checks the whole outcome (status codes, Set-Cookie, revoked row count, both surviving sessions), not just status codes.
- `REFRESH_REUSE_GRACE_SECONDS=0` gives a tested way back to the strict behaviour from the brief.

### Required fixes
- none
