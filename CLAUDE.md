# StreamLine — instructions for Claude

Engineering thesis project (praca inżynierska, Akademia im. Jakuba z Paradyża, 2027):
a web app for Agile project management — Kanban boards in the style of Trello/Jira.
Team of 2: **Nazar Bulyha** and **Andrii Shkraiev**. Both work full-stack, handing work
over through git and the `docs/` folder.

## Privacy rules (IMPORTANT)
- The main repository is shown to the thesis supervisor. **Never mention Claude, AI,
  agents or LLMs** in committed files: code, code comments, `docs/`, `README.md`,
  commit messages, PR titles/descriptions, branch names.
- **Exception — the product's own AI assistant feature** (`docs/REQUIREMENTS.md`, WF-AI, ADR-021,
  milestone M7) is a normal, documented feature: code, docs and commits may describe it
  (e.g. `LlmProvider`, `AI_ENABLED`, "asystent AI"). The ban is only about the tools used to
  *develop* the project (Claude, Claude Code, agents, `.claude/`, `CLAUDE.md`).
- **Never add `Co-Authored-By` trailers or "Generated with ..." lines** to commits or PRs.
  Commits are authored by the human using their own git config.
- `CLAUDE.md` and `.claude/` are NOT tracked by the main repo (excluded via `.git/info/exclude`).
  They live in a separate private config repo with git dir `.git-ai` — see `.claude/AI_WORKFLOW.md`.
  Never add them to the main repo, never add them to `.gitignore` (that would reveal them).
- **`docs/` is also private**: tracked only by the `.git-ai` repo, excluded from the main repo the same way.
  The main repo must never reference `docs/` (no links in README, no "see docs/..." or ADR numbers in
  code comments). Changes to `docs/` are committed to `.git-ai`, not to the main repo.

## Before any work
1. Read `docs/PROJECT_STATE.md` — what is done, in progress, next.
   Scope, roles/permissions matrix and requirement IDs: `docs/REQUIREMENTS.md`.
2. Architecture questions: `docs/ARCHITECTURE.md`, `docs/DECISIONS.md`, `docs/TECH_STACK.md`.
3. Agent workflow: `.claude/AI_WORKFLOW.md`. Human process: `docs/WORKFLOW.md`.

## Stack (details in docs/TECH_STACK.md)
- pnpm monorepo: `apps/web`, `apps/api`, `packages/shared`
- Frontend: React + TypeScript + Vite, React Router, Zustand, @hello-pangea/dnd, socket.io-client, Tailwind + shadcn/ui, react-hook-form, axios
- Backend: Node.js + Express 5 + TypeScript, Prisma + PostgreSQL (Docker), Socket.io, JWT + bcrypt, Zod, pino
- Tests: Vitest, Supertest, React Testing Library, Playwright

## Languages
- Code, identifiers, code comments, commit messages, branch names: **English**.
- Everything in `docs/` and `README.md` (including `docs/thesis-notes/`): **Polish**.
- Talk to the user in the language they write in (usually Ukrainian).

## Code conventions
- TypeScript strict everywhere. No `any` (except justified cases with a comment).
- Backend layers: Controller → Service → Repository. Only repositories touch Prisma.
- Input validation only via Zod schemas from `packages/shared`.
- Ordering of columns and cards only via the `rank` field (LexoRank from `packages/shared`). No integer positions.
- REST = persistent changes; Socket.io = only broadcasting events to room `board:<id>` after a successful REST change.
- Every feature ships with tests. Before finishing: `pnpm lint && pnpm typecheck && pnpm test`.
- Secrets only in `.env` (gitignored); examples in `.env.example`.
- Conventional Commits. Never commit or push without an explicit request from the user.

## Agents (.claude/agents/)
- `coder` — writes code and tests.
- `reviewer` — reviews changes, returns APPROVE or CHANGES_REQUESTED, report goes to `.claude/reviews/`.
- `documenter` — updates `docs/` (Polish).

Full feature cycle: `/feature <task or issue number>`. Before ending a session / handing over: `/handoff`.
