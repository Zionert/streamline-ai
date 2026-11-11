---
name: handoff
description: Prepare the StreamLine repo for handing work over to the teammate — refresh PROJECT_STATE.md and WORKLOG.md, sync the private AI config repo, and produce a short message to send. Use when the user types /handoff or says they are finishing for the day / passing work to a colleague.
argument-hint: [who / comment]
disable-model-invocation: true
---

# /handoff — hand work over to the teammate

User comment: $ARGUMENTS

1. Run `git status -sb`, `git log --oneline -15` and note the current branch and unpushed commits.
2. Call the `documenter` agent: "Handoff mode. Fully refresh `docs/PROJECT_STATE.md`
   (gotowe / w toku / next 3–5 steps by priority / blokery / open questions for the teammate /
   how to run the project from scratch) and add an entry to `docs/WORKLOG.md`."
   Pass it the output from step 1 and the user comment.
3. Main repo: if there are uncommitted changes, show them and ask whether to commit
   (WIP commit on the current feature branch, plain message, no Co-Authored-By) and push.
   Do nothing without permission.
4. AI config repo: run `git --git-dir=.git-ai --work-tree=. status --short`. If `CLAUDE.md` or
   `.claude/` changed, ask whether to commit and push them too
   (`git --git-dir=.git-ai --work-tree=. add CLAUDE.md .claude && ... commit && ... push`).
5. Produce a **ready-to-send message for the teammate** in Ukrainian (5–10 lines):
   branch, what was done, what to do next, what to watch out for,
   "start with `docs/PROJECT_STATE.md`", and remind to pull both repos
   (`git pull` and `git --git-dir=.git-ai --work-tree=. pull`).
