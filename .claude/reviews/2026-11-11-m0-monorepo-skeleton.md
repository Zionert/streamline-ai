# Review — M0 monorepo skeleton (branch feat/m0-monorepo-skeleton)

## Reviewer report — round 1
**Verdict:** APPROVE
**Checks:** lint PASS, typecheck PASS, test PASS (shared 5/5, api 15/15, web 5/5)
Also verified myself: `pnpm format:check` PASS, `pnpm build` PASS (shared -> api -> web, topological),
`docker compose config -q` PASS, built API (`node dist/server.js`) answers `GET /health` -> 200 `{"status":"ok"}`
and unknown route -> 404 in the standard error format. Docker runtime and GitHub CI run still unverified (not possible locally).

### BLOCKER
- none

### MAJOR
- none

### MINOR
- `packages/shared/src/events.ts:5-9` — `WS_EVENTS` already declares `board:updated`, `column:created`, `card:moved`
  although the brief says skeleton only (no models). Harmless, but these names become a de-facto contract before the
  features are designed → either keep it and accept as intentional, or reduce to an empty/minimal object plus
  `boardRoom()` and add events together with the features.
- `apps/api/src/middleware/error-handler.ts:37-39` — every body-parser 4xx (e.g. 413 payload too large, 415) is
  reported with code `BAD_REQUEST`. → derive the code from status (e.g. 413 -> `PAYLOAD_TOO_LARGE`) or name the
  generic branch `CLIENT_ERROR`. Also consider `if (res.headersSent) return next(err);` at the top of the handler
  (standard Express guard for errors after streaming started).
- `.github/workflows/ci.yml:53-54` — "Generate Prisma client" step duplicates the api `postinstall` (`prisma generate`)
  that already runs during `pnpm install`. Not wrong, just redundant → drop it or keep with a comment why it is explicit.
- `.github/workflows/ci.yml:40-47` — action majors `actions/checkout@v7`, `actions/setup-node@v7`,
  `pnpm/action-setup@v6` could not be verified offline (no `gh` available). → confirm on the first CI run; if a tag does
  not exist the workflow fails immediately.
- `package.json` `lint` script — `pnpm -r lint` only lints workspace packages; root files (`eslint.config.js`) are
  covered only by lint-staged on commit. → optional: add a root lint step (`eslint eslint.config.js`) or accept.
- `apps/api/src/server.ts:14` — Prisma client is created in the composition root only to be disconnected on shutdown.
  Fine for a skeleton (it lives in `repositories/db.ts`, so the layering rule holds); when the first repository
  arrives, inject `db` into repositories from here rather than importing a singleton.

### Checked and OK
- Privacy: grep for claude|anthropic|AI|LLM|agent|copilot|gpt|"generated with" over all tracked/untracked files
  (excluding node_modules, .claude, .git-ai, CLAUDE.md, docs/) — no hits; lockfile clean. `CLAUDE.md`, `.claude/`,
  `.git-ai/` excluded via `.git/info/exclude`, not via `.gitignore`. `docs/`, `README.md` untouched.
- Unrelated `cn` package: absent from all `package.json` files, from `pnpm-lock.yaml` and from `node_modules`.
  `cn()` in `apps/web/src/lib/utils.ts` is the correct clsx + tailwind-merge helper.
- Secrets: `.env` / `.env.*` gitignored with `!.env.example`; `apps/api/.env` is ignored; only local-dev/CI default
  credentials (`streamline/streamline`) in examples and CI service — acceptable.
- Security baseline: helmet (x-powered-by removed, nosniff tested), CORS restricted to configured origins (tested
  allowed vs denied), JSON body limit 1 MB, unknown errors hidden behind a generic 500 (tested), env validated with Zod
  (`DATABASE_URL` must be postgres URL, PORT range, CORS list).
- Architecture: routes -> controller -> service for /health; Prisma only in `repositories/db.ts`; `createApp()` with
  injected config/logger keeps the app testable without a server; shared Zod schemas used by both API tests and web hook.
- Windows/Husky: `core.hooksPath=.husky/_`, hook is a plain `pnpm exec lint-staged`; `.gitattributes` forces
  `eol=lf`, so the hook and Prettier `endOfLine: lf` stay consistent on Windows checkouts with `autocrlf=true`.
- Shared package consumption via the `@streamline/source` export condition (sources for dev/test/typecheck/Vite,
  dist for API prod build with `customConditions: []`) — works in all modes I ran.
- @hello-pangea/dnd 18.0.1 with React 19.3.1: smoke test renders DragDropContext/Droppable/Draggable with no
  `console.error`.

### Done well
- Behavioural tests, not "renders": CORS allow/deny, malformed JSON -> 400, 404 format, error mapping incl. ZodError
  and leak protection, env parsing edge cases, web health states incl. schema mismatch and recheck flow.
- Clean, strict TS config (`noUncheckedIndexedAccess`, `verbatimModuleSyntax`), no `any`, type-aware ESLint with
  `no-explicit-any: error` and `consistent-type-imports`.
- Graceful shutdown, lazy DB connection, `/health` deliberately DB-free (liveness), Prisma generate works without
  `DATABASE_URL`.
- CI goes beyond the brief sensibly (format:check + build) with a postgres service and frozen lockfile.
