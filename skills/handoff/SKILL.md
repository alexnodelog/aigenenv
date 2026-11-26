---
name: handoff
description: Write or update state.md, the handoff file for Standard work. Use when a session doing Standard work is about to end or is running low on context; when the human says to stop, pause, wrap up or hand over; when the human approves PRD.md or PLAN.md, or tells you to skip a phase, and that must be recorded; when the orchestrator records a plan or task transition; or when state.md is missing, or disagrees with git, at the start of Standard work. Not for Small tasks, and never from inside a task worktree.
---

# handoff

`state.md` at the repository root is the only memory a new session has, and its reader
knows nothing of this session. Repository facts in it — branches, commits, paths, check
results — come from `git` and the files, never from what the session remembers. Decisions
come from the human, recorded as the human stated them, with the date.

This skill owns the file's format and how it is written. When to record which transition
in an orchestrated run is the `orchestrate` skill's.

## 1. Who writes it

| Situation | Writer | Branch |
|---|---|---|
| A single session doing Standard work | That session | The branch it works on |
| A discover session recording the `PRD.md` approval | That session | The branch tasks will merge into |
| An orchestrated run | The orchestrator only, at every transition, before acting | The branch tasks merge into |

A subagent never writes `state.md`; it reports in its return message. A Small task that
begins and ends in one session has no `state.md` and does not create one.

## 2. Before writing

Commands in this skill are for bash; on Windows they run in WSL2.

1. `git status --porcelain state.md`. If it shows changes this session did not make, stop
   and ask: they are the human's.
2. At the start of Standard work, run §7 before writing anything.
3. Later in the session, when it may end before you finish: if `state.md` is missing, write
   it with the sections of §3 that apply, `## Next` filled with the one action you are on,
   every other section `none`. If it exists, replace only `## Next` with that action. Save
   and commit it (§6), so a session that runs out of room still leaves a valid file.

## 3. Template

```markdown
# State

## Next
<one action a fresh session can take without asking>
Waiting on: <the human: what they must decide | nothing>
Uncommitted: <paths and what they are | none>

## Approvals
- PRD.md @ <full SHA> — YYYY-MM-DD
- PLAN.md @ <full SHA> — YYYY-MM-DD
- PLAN.md @ <full SHA> — YYYY-MM-DD — superseded YYYY-MM-DD
- <file>: skipped by the human — YYYY-MM-DD

## Plan
Status: drafting | awaiting approval | approved | blocked
Target: <the branch tasks merge into>
Needs: <the open decisions | none>

## Tasks
### <task id>
Status: <one task state>
Branch: task/<id>  Worktree: <path relative to the repository root>
Base: <full SHA>  Last commit: <full SHA | none>
Baseline: <command → exit code for each check failing at Base | none>
Review: <verdict> @ <full SHA>, <mode> | none
Findings:
- <severity> <file:line> <what> → <the human's decision: new task | known-issues | pending>
Attempts:
- <cause> / <change> / <observation>
Human:
- YYYY-MM-DD: <answer or direction>
Live checks approved: <check, for which run | none>
Merged: <merge commit SHA>; check: <verdict stands | re-review (other commits | merge edits): verdict> | none

## Exposure
- <risk accepted without a way back, and where it is recorded | none>

## History
- YYYY-MM-DD <what landed> @ <full SHA>; verified by <checks or verdict>
- YYYY-MM-DD task <id> done: approved @ <full SHA>, merged @ <full SHA>; check: <verdict stands | re-review: verdict>
```

- `## Next` is always first and always one action. A list of next steps is a plan, and plans
  live in `PLAN.md`. When the action starts with a skill, `## Next` names it
  ("Start orchestrate in a new session", "Continue discover: <question>").
- Nothing from `PRD.md`, `PLAN.md`, `DECISIONS.md` or a commit is copied in; it is named by
  path, requirement or task id, or SHA.
- Sections that do not apply are left out: a single session with no `PLAN.md` has no
  `## Plan` or `## Tasks`, and records what it just verified in `## History`.
- Task states, exactly these words: `planned`, `spawned`, `built`, `reviewing`, `approved`,
  `changes required`, `unverified`, `blocked`, `struck out`, `merged`, `integration failed`,
  `done`.
- `Baseline` is copied from the first builder's return and never changed afterwards.
- `Attempts` and `Human` keep every entry across sessions; a new session never clears them.
- Only a `done` task is folded: its section is replaced by one `## History` line with the
  task id, the approved commit, the merge commit and the merge-check result. Its medium and
  low findings are decided by then. Every other state keeps its full section.
- A task sent back to `planned` by an amended plan keeps its section; only `Status` changes.
- An approval is never overwritten. A new approval of the same file adds a line, and the
  earlier line is marked `superseded` with the date.
- `## History` is append-only, newest last.
- No secret values anywhere: when the human's answer contains a token, a password or a key,
  record the decision without the value. `## Exposure` describes a risk, never how to use it.

## 4. Gather the facts

Run these now; do not reuse values from earlier in the session.

```bash
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
git status --porcelain
git log --oneline -n 20
git worktree list --porcelain        # orchestrated runs
```

- Every SHA written into `state.md` comes from a command run in this step, in full.
- Committing code is the task's own step, not this skill's. Uncommitted changes, whoever
  made them, are listed under `Uncommitted` with what they are, and are never committed,
  stashed or discarded to make the tree look clean. Uncommitted changes to `state.md`
  itself that this session did not make are the exception: §2 stops first.

## 5. Recording an approval or a skip

The human approves `PRD.md` or `PLAN.md` in the session, in words. Silence is not approval.

1. `git diff --quiet HEAD -- <path>` must exit 0 and the file must be tracked. Otherwise the
   committed version is not the one the human saw: stop and say so.
2. Add its line under `## Approvals`: path, the full SHA from
   `git log -n 1 --format=%H -- <path>`, today's date. An earlier line for the same file is
   kept and marked `superseded` with today's date.
3. When the human tells you to skip a phase, that instruction is the approval: record
   `<file>: skipped by the human — YYYY-MM-DD`.
4. Never write `Approved:` into `PRD.md` or `PLAN.md` itself.
5. Commit `state.md` (§6) before acting on the approval.

Before acting on an approval later, confirm the file is unchanged since that commit:
`git diff --quiet <SHA> -- <path>` exits 0. Any other exit, or no line under `## Approvals`,
means stop and ask.

## 6. Check and commit

1. Read `state.md` back. Every SHA resolves (`git rev-parse --verify --quiet "<SHA>^{commit}"`).
   For each task in `spawned`, `built` or `reviewing`, its worktree appears in
   `git worktree list` and `Last commit` equals `git rev-parse task/<id>`. For each task in
   `approved`, the `Review` SHA equals `Last commit`: a verdict covers one commit only.
   `## Next` is one action.
2. Stage `state.md` alone, by name: `git add state.md`. In an orchestrated run, also
   `known-issues.md` when the human deferred something in this step.
3. `git commit -m "docs(state): <the next action, shortly>"`.
4. Never commit `state.md` on a `task/<id>` branch or in a task worktree.

## 7. When state.md and git disagree

At the start of Standard work, or whenever §6.1 fails:

1. Run §4 and compare each fact with `state.md`: branch, commits, worktrees, uncommitted
   changes.
2. Report every difference to the human. Change nothing — not `state.md`, not the tree.
   Uncommitted changes stay exactly as they are.
3. Approvals and the human's answers are never reconstructed from git history or file
   contents. A missing one is asked for again.
4. Update `state.md` only after the human has decided, and record the decision under
   `Human` with the date.
