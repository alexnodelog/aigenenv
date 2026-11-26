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

Commands are for bash; on Windows they run in WSL2. The checkout stays on the target branch
— the branch tasks merge into — for the whole session. Each subagent's return schema is the
Return section of `.claude/agents/<name>.md`. An agent is "still running" when it was
spawned in this session and its return message has not arrived.

## 1. Start of a session

1. Run skill `handoff` §7. Any difference between `state.md` and git goes to the human
   before anything else.
2. If `## Plan` records a `Target`, the checkout is on it (`git rev-parse --abbrev-ref HEAD`).
   Otherwise stop and ask.
3. Read the check commands from `CLAUDE.md`. If it lists none, stop and ask. Confirm
   `.claude/agents/planner.md`, `builder.md` and `reviewer.md` exist, each with a Return
   section; if not, stop: nothing can be spawned.
4. `PRD.md` has an approval line, and `git diff --quiet <SHA> -- PRD.md` exits 0. Otherwise
   stop: discover comes first.
5. Continue from `state.md`: no `PLAN.md` or plan `drafting` → §2; plan `blocked` → §10;
   `awaiting approval` → §2 step 7; `approved` → confirm `git diff --quiet <SHA> -- PLAN.md`,
   then §10 for every task not `planned` or `done`, then §3.

## 2. Plan

1. Record plan `drafting` and `Target` = the current branch; commit.
2. `git status --porcelain` must be empty. Otherwise stop and ask: the planner works in this
   checkout.
3. Snapshot (§4), then spawn the `planner`. Prompt: nothing for a first plan; when amending,
   the task ids and the reason, and for each amended task that already has a branch, the
   files it has changed (`git diff --name-only <Base> task/<id>`) — the amended task keeps
   each in its file scope, or lists one this task created under files to delete; after
   `BLOCKED`, the human's recorded answer.
4. Check the return (§4).
5. `BLOCKED` → plan `blocked`, `Needs` = its items, commit, and put them to the human (§11).
   - An item starting `PRD:` belongs in `PRD.md`: tell the human to answer it in a discover
     session, and stop this session.
   - Otherwise record the answer under the plan, set `drafting`, and go to step 3.
6. `DONE` → `PLAN.md` has at least one task; every task id matches `^[a-z0-9][a-z0-9-]*$`
   and appears once; every path under file scope and files to delete is relative to the
   repository root, with no `..` component. If not, do not commit `PLAN.md`; respawn the
   planner with that as the reason. Then commit `PLAN.md` alone
   (`git add PLAN.md`, `git commit -m "docs(plan): <what this version plans>"`), set
   `awaiting approval`, commit `state.md`, and ask the human to approve `PLAN.md` at that
   short SHA. Point them to what they approve: tasks and order, file scopes, files to
   delete, review modes, `parallel with` pairs, live checks.
7. An unconditional yes → record it (handoff §5), plan `approved`, commit. On the first
   approval every task becomes `planned`. On an amended plan only new and amended tasks
   become `planned`; every other task keeps its state. A conditional or partial yes ("except task 4", "but keep that file") is a request
   for changes: `drafting`, and step 3 with it as the reason.

An approved plan that needs amending goes back to step 1 with the task ids. An amended task
that already has a branch keeps it: once `planned` again it continues there through
§3 Respawn, not as a new task.

## 3. Build

No new task starts while any task is `merged` with its check not yet passed, `unverified`
after its merge, or `integration failed` — except the fix tasks recorded for that
integration failure (§8). Otherwise take the next `planned` task whose `blocked by` tasks are all
`done`. If it and another such task name each other under `parallel with`, confirm from
`PLAN.md` that neither blocks the other, directly or through other tasks, and that their file
scopes share no path (a directory covers every path under it). If so, start both — each
through this section — before waiting on either; if not, run them one at a time and tell the
human. Nothing else allows two at once.

Never spawn an agent for a task while another agent for that task may still be running. A
spawn that errors or returns nothing usable has an unknown outcome: go to §10, not to a
second spawn.

**New task** (`planned`):

1. Ask the human about this task's live checks, one decision per check. Record the ones
   approved for this run. A `human-run` check is never approved for an agent.
2. Record `spawned`, branch `task/<id>`, worktree `../<repo>.worktrees/<id>`,
   `Base` = `git rev-parse HEAD`. Commit `state.md`.
3. `git rev-parse --verify --quiet refs/heads/task/<id>` must fail and the path must not
   exist. If `state.md` records that branch for this task — an amended task — use Respawn
   below instead; any other existing branch or path → §10. Then
   `git worktree add -b task/<id> <absolute path> <Base>`, and confirm its HEAD equals
   `Base` and its branch is `task/<id>`. If the command fails or either does not hold → §10.
4. Spawn the builder (below).

**Respawn** (`changes required`; `blocked` / `struck out` with the human's answer recorded;
an amended task that has a branch):

1. The worktree is listed by `git worktree list` on branch `task/<id>`, its HEAD equals the
   recorded `Last commit` (or `Base` if none), and its `git status --porcelain` is empty.
   Anything else → §10.
2. Ask about live checks again for the new run, as above.
3. Record `spawned`, commit. Spawn the builder (below).

**Spawning the builder.** Snapshot (§4), then spawn the `builder`. Prompt:

- always: the task text from `PLAN.md`, its requirement from `PRD.md` quoted as written,
  the file scope, the absolute worktree path, the check commands, `Base`;
- rework: the HEAD of `task/<id>` and the reviewer's findings from `state.md`;
- any respawn: the recorded `Baseline`;
- after `BLOCKED` or `STRUCK_OUT`: every `Human` entry and every attempt;
- an amended task: the HEAD of `task/<id>`, and that the commits after `Base` were made for
  the earlier version of this task — keep what the amended task needs, change what it does
  not;
- the live checks approved for this run.

Never the previous builder's message or a summary of earlier sessions.

## 4. Snapshot and return check

**Before every spawn**, note for the main checkout and every worktree under
`../<repo>.worktrees/`: HEAD and `git status --porcelain`. Other worktrees of the human's are
not compared. The snapshot lives only
in this session, for the return check. If the session ends before the return, §10 applies.

**After every return**, before recording anything:

1. The message has every field its schema requires and nothing else.
2. Role checks:

   | Role | Must hold |
   |---|---|
   | planner | `git status --porcelain` in the main checkout lists `PLAN.md` and nothing else — staged, unstaged and untracked alike |
   | builder | The worktree is on branch `task/<id>` (`git -C <worktree> symbolic-ref --short HEAD`) and its HEAD equals `git rev-parse task/<id>`; `COMMIT` equals that HEAD — `none` is accepted only with `BLOCKED`, and then the branch must still be at the recorded `Last commit` (or `Base`); `BASE` equals the recorded `Base`, and `git merge-base --is-ancestor <Base> task/<id>` succeeds; on a respawn `BASELINE` equals the recorded one; on `DONE`, `git -C <worktree> status --porcelain` is empty |
   | reviewer | both `COMMIT` SHAs equal the head it was given; `TREE: clean`; the review worktree's `git status --porcelain` is empty; an `approved` lists no critical or high finding unless it is marked `pre-existing` |

3. Every other worktree, and the main checkout, matches its snapshot — except the
   worktrees of agents still running in a parallel pair, which are not compared, and
   `PLAN.md` for the planner.

Any failure: reviewer → `unverified`; planner or builder → `BLOCKED`. Record what failed and
take it to the human. Never fill in a missing field or fix the tree.

## 5. Builder return

- `DONE` → record `Last commit`, `Baseline` (first builder only), checks, `NOT REACHED`;
  task `built`; commit. Then §6.
- `BLOCKED` → task `blocked`, record `NEEDS`, commit, take it to the human. If the answer
  changes the task's requirement, file scope, files to delete, seams under test, done condition or
  `blocked by`, the plan is amended (§2) and the task is `planned` again when that is
  approved. If it only changes how the task is done, record it under `Human` and respawn
  (§3).
- `STRUCK_OUT` → task `struck out`, record every attempt, commit, take it to the human.
  Respawn (§3) only with their direction recorded.
- `NOTES` → put each item to the human (§11) with what it affects and a suggestion: a
  Notify-tier change (`GLOBAL_RULES.md` §4) is reported; an unrelated problem or a known-issues proposal becomes a
  new task (plan amendment), a line in `known-issues.md`, or nothing. Record the decision.

The worktree of a blocked or struck-out task is kept. Uncommitted changes in it are the
human's to commit or discard. If the human commits there and says to keep the commit, record
it as `Last commit` with a `Human` entry; the next builder starts from it.

## 6. Review

1. Record task `reviewing`, commit.
2. If `../<repo>.worktrees/review-<id>-<short SHA>` is left from an earlier review: remove
   it when its `git status --porcelain` is empty; otherwise stop — the human clears it. Then
   `git worktree add --detach <that path> <Last commit>`; confirm its HEAD.
3. Snapshot (§4), then spawn the `reviewer`. Prompt: the task text, base `Base` and head
   `Last commit`, the mode from `PLAN.md`, the check commands, the review worktree's absolute
   path, the live checks approved for this run. Never the builder's message.
4. Check the return (§4). Remove the review worktree only when its `git status --porcelain`
   is empty (`git worktree remove <path>`, never `--force`).

## 7. Verdict

Record the verdict with its commit and mode, `NOT REACHED`, and every finding; commit.

- `changes required` → task `changes required`; respawn (§3) as rework.
- `unverified` → put the reason to the human: review again (`reviewing`, §6) or treat as
  `blocked` (§5).
- `approved` → for each finding — medium, low, or `pre-existing` of any severity — the
  human decides: a new task (plan amendment) or a line in `known-issues.md`. Record each
  decision. Then §8.

`known-issues.md` lines: `- YYYY-MM-DD <issue> — <why deferred> (task <id>, <severity>)`.

## 8. Merge

**Before handing over the merge:**

1. The checkout is on `Target`, and `git rev-parse -q --verify MERGE_HEAD` fails (no merge
   in progress).
2. `git rev-parse task/<id>` equals the approved commit. If the branch has moved, stop:
   report the commits after the approved one. The verdict covers only that commit; the
   human decides.
3. Give the human the command. The first `-m` is theirs to word; `--no-ff` and the
   `Reviewed:` line are required:

   ```bash
   git merge --no-ff task/<id> -m "merge: task <id> <title>" \
     -m "Reviewed: approved @ <approved commit> (<mode>)"
   ```

**If the merge stops on a conflict** — the human says so, or `MERGE_HEAD` exists: touch
nothing. The task stays `approved`. The human either resolves and commits the merge (the
merge check then finds the hand edits and §8 step 4 reviews them), or runs
`git merge --abort`. Continue when they say which.

**When they say it is merged:**

1. The checkout is on `Target`. Find the merge: the lines of
   `git rev-list --first-parent --merges --parents <Base>..HEAD` (merge, first parent,
   second parent) whose second parent is the approved commit. Later commits on `Target` do
   not hide it. Exactly one → that is `<merge>`. None or more than one → stop: the merge
   check cannot run, and the human decides.
2. Record task `merged`, `Merged: <merge>`, commit.
3. Merge check, with `state.md` and `known-issues.md` excluded. First
   `git merge-base --is-ancestor <Base> <merge>^1` must succeed; if it does not, the target
   branch was rewritten — the verdict cannot stand on its own, so go to step 4's re-review.
   - Condition 1: `git log --format= --name-only <Base>..<merge>^1` lists no path other
     than those two.
   - Condition 2: `git diff --quiet <approved> <merge> -- . ':(exclude)state.md' ':(exclude)known-issues.md'`
     exits 0.

   For the second task of a parallel pair, condition 1 fails because the first task is
   already on `Target`. That is expected: the check-mode review below is the pair's
   integration review.
4. Both hold → `check: verdict stands`, task `done`. Either fails → record which, and spawn
   a `check`-mode reviewer on the merge commit (§6, base `<merge>^1`, head `<merge>`). Its
   verdict: `approved` → `done`; `unverified` → task `unverified`; `changes required` →
   `integration failed`. The human adds one or more fix tasks by plan amendment (§2); record
   their ids under this task's `Human` as `fix: <ids>`. Only they may start (§3). When every
   recorded fix task is `done`, this task is `done`.
5. On `done`: remove the task worktree if its `git status --porcelain` is empty
   (`git worktree remove`, never `--force`). Branches are never deleted by an agent. Fold the
   task with skill `handoff`.

## 9. Ending the session

One session takes one task — or one parallel pair — to `done`. When a task reaches `done`
and every other task started in this session is `done` or waiting on the human (`blocked`,
`struck out`, `unverified`, `integration failed`), set `## Next` to the next action, commit, and
tell the human to end this session and start a new one with `orchestrate`. Never end it
while a subagent is still running.

## 10. Recovery

Recovery goes by what git shows, never by what `state.md` says should have happened. Before
redoing a spawn, establish that no agent for that task is still running — the session that
spawned it has ended, or the human confirms it. An operation whose outcome is unknown is
never simply repeated.

| Recorded state | Check | Allowed next step |
|---|---|---|
| `spawned`, no branch or worktree | `task/<id>` and the path are both absent | New task, §3 step 3 |
| `spawned` | Worktree on `task/<id>`, HEAD equals `Last commit` (or `Base`), clean | Respawn with the same prompt |
| `built` | Branch HEAD equals `Last commit` | §6 |
| `reviewing` | Branch HEAD equals `Last commit`; a leftover review worktree is clean | Remove it if clean, then §6 again — a reviewer writes nothing, so a review can be redone |
| `approved` | §8 "When they say it is merged" step 1 finds exactly one merge | The merge happened: continue there |
| `approved` | `MERGE_HEAD` exists | §8 conflict |
| `approved` | Neither | Hand over the merge again (§8) |
| `merged` | — | Run the merge check again; it only reads |
| plan `blocked` | The human's answer to `Needs` is recorded after the block | §2 step 3 (or discover, for a `PRD:` item) |
| plan `blocked` | No answer recorded | Put `Needs` to the human (§11) |
| `changes required` | As for Respawn (§3) | Respawn as rework |
| `blocked`, `struck out` | The human's answer is recorded after the stop | §5 for that answer: respawn, or plan amendment |
| `blocked`, `struck out` | No answer recorded | Put `NEEDS` or the attempts to the human (§11) |
| `unverified` | — | Ask the human: review again (§6) or treat as `blocked` |
| `integration failed` | The fix tasks recorded under `Human` | Continue them; if none is recorded, ask the human to add one |

Anything else — new commits, uncommitted changes, a branch or path `state.md` does not
record — is reported exactly as found, and the human decides what is kept. Never reset,
discard, stash, force-remove or delete to make recovery possible. A live check is never
repeated without the human's approval for the new run.

## 11. Putting a decision to the human

Each decision is one question, in the human's language:

```
<what is undecided, in one line> — <why it needs them>
A. <option> — <what happens next>
B. <option> — <what happens next>
Suggested: <A or B>, because <reason>   (only when it rests on something stated)
```

Name the task id and, where it helps, the file or finding. Do not paste return messages
or logs; quote only the line that matters. Record the answer under `Human` with the date
before acting on it.
