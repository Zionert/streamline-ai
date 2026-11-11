---
name: feature
description: Run the full StreamLine development cycle for one task — coder implements, reviewer checks (loop until APPROVE, max 3 rounds), documenter updates docs. Use when the user types /feature or asks to implement a task "through the agents".
argument-hint: <task description or issue number>
disable-model-invocation: true
---

# /feature — full task cycle

Task: **$ARGUMENTS**

You are the orchestrator. You do not write code yourself — you delegate to agents via the Agent tool
and pass results between them. Keep the user posted with one line per step (in the user's language).

## 0. Preparation
- If `$ARGUMENTS` is a GitHub issue number, get its description (`gh issue view <N>` if `gh` is
  available; otherwise ask the user to paste it).
- Read `docs/PROJECT_STATE.md`. If the task is unclear or conflicts with the architecture,
  ask the user 1–3 clarifying questions first.
- Check `git status` and the current branch. If on `main`, propose a branch name
  (`feat/<issue>-<slug>`) and create it after the user agrees. If there are unrelated
  uncommitted changes, ask what to do with them.
- Write a clear brief for the coder: what to do, acceptance criteria, what NOT to do.

## 1. Coder
Call the `coder` agent with the brief. Keep its report.

## 2. Reviewer (loop)
Call the `reviewer` agent with: brief + coder report + round number.
- **APPROVE** → step 3.
- **CHANGES_REQUESTED** → send the reviewer report to the `coder` (continue the same coder via
  SendMessage if possible, otherwise a new call with brief + report) and review again.
- Max **3 rounds**. If still CHANGES_REQUESTED after round 3, stop and show the user the unresolved
  items — the human decides.

## 3. Documenter
Call the `documenter` agent with: brief + final coder report + final reviewer report.

## 4. Summary for the user
- What was done (2–4 bullets), number of review rounds.
- How to verify manually (commands / what to click).
- Proposed commit message (Conventional Commits, English, **no Co-Authored-By, no AI mention**).
- **Do not commit on your own** — ask for permission. When committing, use the user's git identity
  as is and a plain message.
- On permission, commit to **both** repos:
  1. main repo: code only (`docs/`, `CLAUDE.md`, `.claude/` are excluded automatically);
  2. private repo: `git --git-dir=.git-ai --work-tree=. add docs CLAUDE.md .claude` →
     commit (e.g. `docs: <task>`) → push when the user asks to push.
