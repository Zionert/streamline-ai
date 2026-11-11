# AI workflow (private — not part of the main repo)

This file, `CLAUDE.md` and everything in `.claude/` live in a **separate private repo**
(git dir `.git-ai`) that shares the same working folder as the main repo.
The main repo never sees these files.

## Two repos, one folder

```
streamline/                 ← working folder
├─ .git/                    ← MAIN repo (GitHub: streamline) — code + docs (Polish)
├─ .git-ai/                 ← AI CONFIG repo (GitHub: streamline-ai, private) — CLAUDE.md + .claude/
├─ CLAUDE.md                ┐ tracked only by .git-ai
├─ .claude/                 ┘ (main repo ignores them via .git/info/exclude — NOT .gitignore)
├─ apps/ packages/ docs/ …  ← tracked only by .git
```

- Main repo: `.git/info/exclude` contains `CLAUDE.md`, `.claude/`, `.git-ai/`.
  This file is local and never pushed, so nothing in the main repo hints at Claude.
- AI repo: `.git-ai/info/exclude` contains `/*` plus `!/CLAUDE.md` and `!/.claude/`,
  so it tracks only the AI files.

### Everyday commands
```bash
git status                                   # main repo, as usual
git --git-dir=.git-ai --work-tree=. status   # AI config repo
git --git-dir=.git-ai --work-tree=. add CLAUDE.md .claude
git --git-dir=.git-ai --work-tree=. commit -m "update agents"
git --git-dir=.git-ai --work-tree=. push
```
Tip: add an alias once: `git config --global alias.ai '!git --git-dir=.git-ai --work-tree=.'`
→ then `git ai status`, `git ai pull`, `git ai push`.

### Setup for the teammate (one time)
```bash
git clone https://github.com/Zionert/streamline.git
cd streamline
git clone --bare https://github.com/Zionert/streamline-ai.git .git-ai
git --git-dir=.git-ai config core.bare false
git --git-dir=.git-ai config core.worktree ..
git --git-dir=.git-ai --work-tree=. checkout -f main
printf '/*\n!/CLAUDE.md\n!/.claude/\n' > .git-ai/info/exclude
printf 'CLAUDE.md\n.claude/\n.git-ai/\n' >> .git/info/exclude
```
Then check: `git status` must NOT show `CLAUDE.md` / `.claude/`.

## Commit rules
- Commits are authored with the person's own `git config user.name / user.email`.
- No `Co-Authored-By`, no "Generated with", no AI mentions in messages, PRs, branch names,
  code comments or `docs/`. Also enforced by `.claude/settings.json` (attribution disabled).

## Agent cycle for one task

```
 /feature #12
   │  orchestrator (main session): reads issue + PROJECT_STATE, writes brief, creates branch
   ▼
 coder ──report──► reviewer ──APPROVE──► documenter ──► summary + proposed commit message
   ▲                  │
   └──fix requests────┘   (max 3 rounds, then the human decides)
```

| Agent | Does | Does not | Model |
|---|---|---|---|
| `coder` | code + tests, runs lint/typecheck/test, reports | commit, edit `docs/` | inherit |
| `reviewer` | bugs, security, architecture, tests, privacy; report → `.claude/reviews/` | fix code | inherit |
| `documenter` | updates `docs/` in Polish from the real diff | touch code | sonnet |

### Human steps per task
1. Take an issue, move it to In Progress.
2. `git checkout main && git pull`
3. In Claude: `/feature #12` (or `/feature <description>`). Answer clarifying questions.
4. Read the summary, **test it by hand** (`pnpm dev`), read the diff — you must be able to explain the code at the defense.
5. Allow the commit → push → open PR on GitHub → teammate reviews and merges.

Small things (typo, rename): just ask Claude directly, no full cycle; still ask it to add a WORKLOG entry.

### Handoff
- Finishing: `/handoff` → updates PROJECT_STATE + WORKLOG, offers to commit/push both repos,
  generates a message for the teammate.
- Starting: `git pull && git ai pull && pnpm install`, then ask Claude:
  "Read docs/PROJECT_STATE.md and the latest WORKLOG entries, summarize where we stopped and what's next."
