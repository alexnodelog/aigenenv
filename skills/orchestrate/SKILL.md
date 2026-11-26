---
name: orchestrate
description: Run Standard work from Plan through Verify in the main session — spawn the planner, builders and reviewers, check every return against git, and keep state.md. Use when state.md's Next says to start or continue orchestrate; when PRD.md is approved and PLAN.md is missing or not approved; when a task in state.md is planned, spawned, built, reviewing, approved, merged, blocked, struck out, unverified or integration failed; or when the human says they merged a task branch. Not for Small tasks, and never inside a subagent.
---

# orchestrate

The orchestrator is the main session. It is the only role that spawns, the only writer of
`state.md` and `known-issues.md` while it runs, and it edits nothing else: no code, tests,
`PRD.md` or `PLAN.md`, and it never merges. It reads `PRD.md`, `PLAN.md`, `state.md` and the
subagents' return messages; it does not read code. `state.md` is written with skill
`handoff`, at every transition, and committed before the step it records is taken.

Commands are for bash; on Windows they run in WSL2. The checkout stays on the branch tasks
merge into for the whole session.

## 1. Start of a session

1. Run skill `handoff` §7. Any difference between `state.md` and git goes to the human
   before anything else.
2. Read the check commands from `CLAUDE.md`. If it lists none, stop and ask: builders and
   reviewers cannot run without them.
3. `PRD.md` has an approval line, and `git diff --quiet <SHA> -- PRD.md` exits 0. Otherwise
   stop: discover comes first.
4. Continue from what `state.md` records: no `PLAN.md` or plan `drafting` → §2; plan
   `awaiting approval` → §2 step 5; plan `approved` → §1 step 5; a task left `spawned` or
   `reviewing` → §10.
5. With the plan approved, confirm `PLAN.md` is unchanged since its approval line
   (`git diff --quiet <SHA> -- PLAN.md`), then act on the task states in §3 onward.

## 2. Plan

1. Record plan `drafting` (handoff), commit.
2. Snapshot (§4), then spawn the `planner`. Prompt: nothing for a first plan; when amending,
   the task ids and the reason; after `BLOCKED`, the human's recorded answer.
3. Check the return (§4). Planner: the message is exactly its schema, and `PLAN.md` is the
   only changed path in the checkout.
4. `BLOCKED` → plan `blocked`, `Needs` = its items, commit, and put them to the human (§11).
   - A `NEEDS` item starting `PRD:` belongs in `PRD.md`: tell the human to answer it in a
     discover session, and stop this session.
   - Otherwise, record the answer under the plan, set `drafting`, and spawn again.
5. `DONE` → commit `PLAN.md` alone (`git add PLAN.md`,
   `git commit -m "docs(plan): <what this version plans>"`), set `awaiting approval`, commit
   `state.md`, and ask the human to approve `PLAN.md` at that short SHA. Point them to what
   they are approving: tasks and their order, file scopes, files to delete, review modes,
   `parallel with` pairs, and live checks.
6. Approved → record it (handoff §5), plan `approved`, every task `planned`, commit.
   Changes asked for → `drafting`, and spawn the planner with the change as the reason.

An approved plan that needs amending goes back to step 1 with the task ids. The amended
tasks are `planned` again once the new version is approved; the old approval line is kept
as superseded.

## 3. Build

Take the next `planned` task whose `blocked by` tasks are all `done`. Two tasks run at once
only when each names the other under `parallel with`; nothing else allows it.

1. Ask the human about this task's live checks, one decision per check, if it has any.
   Record the ones approved for this run. A `human-run` check is never approved for an agent.
2. Record the task: `spawned`, branch `task/<id>`, worktree
   `../<repo>.worktrees/<id>`, `Base` = `git rev-parse HEAD` now. Commit `state.md`.
3. Create the worktree at `Base`:
   - `git rev-parse --verify --quiet refs/heads/task/<id>` must fail and the path must not
     exist. If either exists, §10.
   - `git worktree add -b task/<id> <absolute path> <Base>`.
   - Confirm `git -C <path> rev-parse HEAD` equals `Base` and the branch is `task/<id>`.
4. Snapshot (§4), then spawn the `builder`. Prompt:
   - always: the task text from `PLAN.md`, its requirement from `PRD.md` quoted as written,
     the file scope, the absolute worktree path, the check commands, `Base`;
   - rework: the HEAD of `task/<id>` and the reviewer's findings from `state.md`;
   - any respawn: the recorded `Baseline`;
   - after `BLOCKED` or `STRUCK_OUT`: every `Human` entry and every attempt;
   - the live checks approved for this run.
   Never the previous builder's message or a summary of earlier sessions.

## 4. Snapshot and return check

**Before every spawn**, record for the main checkout and every task worktree
(`git worktree list --porcelain`): HEAD and `git status --porcelain`.

**After every return**, before recording anything:

1. The message has every field its schema requires and nothing else.
2. Role checks:

   | Role | Must hold |
   |---|---|
   | planner | `PLAN.md` is the only changed path in the checkout |
   | builder | `COMMIT` exists and equals `git rev-parse task/<id>`; `BASE` equals the recorded `Base`; on a respawn `BASELINE` equals the recorded one; on `DONE`, `git -C <worktree> status --porcelain` is empty |
   | reviewer | both `COMMIT` SHAs equal the head it was given; `TREE: clean`; the review worktree's `git status --porcelain` is empty |

3. Everything outside the spawned agent's own worktree has the HEAD and status recorded in
   the snapshot. For the planner, which works in the main checkout, the only allowed
   difference is `PLAN.md`.

Any failure: reviewer → `unverified`; planner or builder → `BLOCKED`. Record what failed and
take it to the human. Never fill in a missing field or fix the tree.

## 5. Builder return

- `DONE` → record `Last commit`, `Baseline` (first builder only), checks, `NOT REACHED`;
  task `built`; commit. Then §6.
- `BLOCKED` → task `blocked`, record `NEEDS`, commit, take it to the human. The answer is
  either new direction for the same task — record it under `Human`, task `spawned`, §3 step
  4 — or a change to the task (a file outside its scope, an unworkable seam) — the plan is
  amended (§2) and the task is `planned` again when that is approved.
- `STRUCK_OUT` → task `struck out`, record every attempt, commit, take it to the human.
  Respawn only with their direction recorded.
- `NOTES` → put each item to the human: a notify-tier change is reported; an unrelated
  problem or a known-issues proposal becomes a new task (plan amendment), a line in
  `known-issues.md`, or nothing, as the human decides.

The worktree of a blocked or struck-out task is kept. Uncommitted changes in it are the
human's to commit or discard.

## 6. Review

1. Record task `reviewing`, commit.
2. `git worktree add --detach ../<repo>.worktrees/review-<id>-<short SHA> <Last commit>`;
   confirm its HEAD.
3. Snapshot (§4), then spawn the `reviewer`. Prompt: the task text, base `Base` and head
   `Last commit`, the mode from `PLAN.md`, the check commands, the review worktree's absolute
   path, the live checks approved for this run. Never the builder's message.
4. Check the return (§4). Remove the review worktree only when its `git status --porcelain`
   is empty (`git worktree remove <path>`, never `--force`).

## 7. Verdict

Record the verdict with its commit and mode, `NOT REACHED`, and every finding; commit.

- `changes required` → task `changes required`, then `spawned` for a fresh builder (§3
  step 4, rework).
- `unverified` → put the reason to the human: review again (`reviewing`, §6) or treat as
  `blocked` (§5).
- `approved` → for each medium or low finding, the human decides: a new task (plan
  amendment) or a line in `known-issues.md`. Record each decision. Then hand over the merge
  (§8).

`known-issues.md` lines: `- YYYY-MM-DD <issue> — <why deferred> (task <id>, <severity>)`.

## 8. Merge

The human merges; the orchestrator never does. Give them the exact command, run from the
checkout on the target branch:

```bash
git merge --no-ff task/<id> -m "merge: task <id> <one-line title>" \
  -m "Reviewed: approved @ <Last commit> (<mode>)"
```

When they say it is merged:

1. `git rev-parse HEAD^2` equals the approved commit. If HEAD is not a merge commit, or its
   second parent differs, stop: the merge check cannot run, and the human decides.
2. Record task `merged`, `Merged: <HEAD>`, commit.
3. Merge check, with `state.md` and `known-issues.md` excluded:
   - Condition 1: `git log --format= --name-only <Base>..<merge>^1` lists no path other
     than those two.
   - Condition 2: `git diff --quiet <approved> <merge> -- . ':(exclude)state.md' ':(exclude)known-issues.md'`
     exits 0.
4. Both hold → `check: verdict stands`, task `done`.
   Either fails → spawn a `check`-mode reviewer on the merge commit (§6, base `<merge>^1`,
   head `<merge>`), recording which condition failed. Its verdict:
   `approved` → `done`; `unverified` → task `unverified`; `changes required` → task
   `integration failed` — the human adds a fix task, and the task is `done` only when that
   fix is merged and the check-mode review of its merge commit approves.
5. On `done`: remove the task worktree if `git status --porcelain` there is empty
   (`git worktree remove`, never `--force`). Branches are never deleted by an agent. Fold the
   task with skill `handoff`.

No dependent task starts before its blockers are `done`.

## 9. Ending the session

When a task reaches `done` and no other task is `spawned` or `reviewing`, set `## Next` to
the next action, commit, and tell the human to end this session and start a new one with
`orchestrate`. Do not end it while a subagent is still running.

## 10. Resuming a task left mid-step

For a task left `spawned` or `reviewing`, or whose branch or worktree already exists:

- No commits on the branch past the recorded `Last commit` (or `Base`) and a clean worktree
  → redo the step.
- Anything else — new commits, uncommitted changes, a branch or path `state.md` does not
  record → report exactly what is there and stop. The human decides what is kept.
- Never reset, discard, stash, force-remove or delete to make resuming possible. A live
  check is never repeated on resume without the human's approval for the new run.

## 11. Putting a decision to the human

The human may not be a developer. Each decision is one question, in their language:

```
<what is undecided, in one line> — <why it needs them>
A. <option> — <what happens next>
B. <option> — <what happens next>
```

Name the task id and, where it helps, the file or finding. Do not paste return messages
or logs; quote only the line that matters. Record the answer under `Human` with the date
before acting on it.
