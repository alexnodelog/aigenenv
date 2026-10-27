# AGENT_DESIGN.md

Which agents exist, where each one's work starts and stops, and how they hand work to each
other. This document is **not deployed**. Like `TECH_STACK.md`, it is the source from which
deployed files are derived: `agents/*.md` (agent definitions) and the orchestration skills.
Edit here first, then regenerate the derived files. Deployed files MUST NOT reference this
document.

Precedence is unchanged: `GLOBAL_RULES.md → rules/<project>.md → CLAUDE.md → current task`.
An agent definition adds role-specific behaviour; it never relaxes a rule.

**Precondition.** This design conflicts with the current `GLOBAL_RULES.md` until the changes
in §9 are made there. No file derived from this document is deployed before those changes
are made, every skill an agent definition names exists, the way `CLAUDE.md` brings the
rules into a subagent has been verified on the installed CLI, and the human has confirmed
that the machine agents run on holds no production credentials (§3).

---

## 1. Facts this design depends on

Checked 2026-10-26 against the Claude Code subagent docs
(`https://code.claude.com/docs/en/sub-agents`) and the Codex subagent docs
(`https://developers.openai.com/codex/subagents`). Re-check against the docs and the installed
CLI version before changing a decision that rests on one.

| Fact | Consequence |
|---|---|
| A subagent cannot hold a dialogue with the human. It receives one prompt and returns one final message | Any phase that is mostly asking (Discover) cannot be a subagent. A subagent that needs a decision stops and returns |
| The subagent's final message is the only channel back to the parent | That message carries status only, in a fixed shape (§5). Anything needed later is persisted to a file by the parent before it acts |
| Claude Code custom subagents load `CLAUDE.md` and the git status; their context otherwise starts from the definition body and the prompt | Agent definitions MUST NOT restate global or archetype rules. `CLAUDE.md` is what brings those rules into every agent |
| Nesting: the Claude Code docs say subagents cannot spawn subagents; the v2.1.172 changelog says they can, up to 5 levels | Unresolved. The design MUST NOT depend on nesting: only the main session spawns |
| Definition formats differ by CLI: Claude Code uses Markdown + YAML in `.claude/agents/`; Codex uses TOML in `.codex/agents/` (`name`, `description`, `developer_instructions`) | One canonical Markdown form (§6). Other CLIs need a conversion, which is out of scope until one is adopted |

---

## 2. Roster

Three subagents, two main-session skills. Nothing else is an agent.

| Role | Runs as | Phase | Reads | Writes | Never |
|---|---|---|---|---|---|
| `discover` | Skill, in an interactive session with the human | Discover | The request, existing `PRD.md`, code only to answer a factual question | `PRD.md`; its approval record in `state.md` | Writes code or `PLAN.md`; leaves an open question in `PRD.md` |
| `orchestrate` | Skill, in the main session | Plan → Verify | `PRD.md`, `PLAN.md`, `state.md`, subagent return messages | `state.md`, `known-issues.md` | Edits code, tests, `PRD.md`, `PLAN.md`; forwards a builder's message to the reviewer; merges |
| `planner` | Subagent | Plan | `PRD.md`, the code it needs to size tasks | `PLAN.md` | Writes code; adds a task with no requirement behind it |
| `builder` | Subagent, in a task worktree | Build | One `PLAN.md` task, its requirement, the code in scope | Code, tests, `DECISIONS.md`, `README.md`, `.env.example` — on its task branch | Touches `state.md`; widens scope; starts another task |
| `reviewer` | Subagent, in a detached worktree at the commit under review | Verify | The task text, the diff range, the check commands, the repository at that commit | Nothing in the repository | Reads `state.md` or `known-issues.md`; fixes what it finds |

Supporting skills, loaded only when used: `handoff` (closing a session into `state.md`),
`diagnose` (a bug whose cause is unknown; used by `builder` and by single sessions).

**Small tasks use none of this.** One session does the task and the human reads the diff.

### Why each boundary is where it is

- **Discover is not a subagent** because it is a conversation. Running it in the
  orchestrator's session would fill that session with the interview, so it gets its own
  session, which closes once `PRD.md` is approved.
- **The orchestrator is a skill, not an agent definition**, because it is the only role
  that runs in the main session, and a skill is the mechanism that loads role instructions
  into the main session on demand.
- **Planner is a subagent** because sizing tasks requires reading code, and that reading
  should not land in the orchestrator's context.
- **Debugging is not a separate agent.** A bug fix is already a builder task that starts
  with a reproducing test; `diagnose` is the procedure the builder follows when the cause is
  unknown. A second agent would add an escalation path the three-strikes rule forbids.
- **One reviewer with two modes**, not two agents. The isolation rules, inputs, and verdict
  are identical; only the depth differs. Two files would duplicate the part most likely to
  drift.

---

## 3. Flow of one Standard task

```
human ──▶ [discover session] ──▶ PRD.md ──▶ human approves
                                              │
          ┌───────────────────────────────────┘
          ▼
[orchestrate] ──spawn──▶ planner ──▶ PLAN.md ──▶ human approves
     │
     ├─ per task, in PLAN.md order (parallel only for tasks marked parallel: yes):
     │     record task as spawned (branch, worktree, base commit) in state.md
     │     create worktree + branch task/<id> at the base commit
     │     spawn builder ──▶ commits on task/<id>, returns status
     │     verify the return against git, record it in state.md
     │     create detached worktree at the returned commit
     │     spawn reviewer (mode by task) ──▶ returns verdict
     │     verify nothing outside the review worktree changed, record verdict
     │       approved            → human merges task/<id>, then the merge check (below)
     │       changes required    → findings into state.md, spawn a fresh builder
     │       unverified          → human decides
     │
     ├─ builder BLOCKED / STRUCK_OUT → ask the human, record the answer, respawn a builder
     └─ planner BLOCKED              → ask the human, record the answer, respawn the planner
```

**One task, one human checkpoint.** Every task ends with a reviewer verdict and the human
merging its branch. The next task that depends on it starts only after that merge. This is a
chosen cost of the Human-in-the-Loop policy, not a safety requirement of every change.

**Sequential or parallel.** Tasks run one at a time by default. Two tasks MAY run at the same
time only when `PLAN.md` marks both `parallel: yes`, which the planner may do only when
neither blocks the other and their file scopes do not overlap. The human approves this with
the plan; the orchestrator never decides it on its own. Both modes use the same procedure —
a worktree and a `task/<id>` branch per task — so switching between them changes nothing
else.

**Reviewer mode.** `adversarial` when the task is in the Always-Standard list or changes a
public interface; `check` for every other Standard task. The orchestrator states the mode
in the spawn prompt; the reviewer does not choose it.

**Merge check.** A verdict covers one commit, not the merge. After the human merges, the
orchestrator lets the verdict stand only when both hold, with `state.md` and
`known-issues.md` excluded throughout:

1. Every commit on the target branch between `BASE` and the merge commit's first parent
   changes only those two files — no other code arrived while the task was open.
2. `git diff <approved commit> <merge commit>` is empty — the merge brought in exactly the
   approved tree, with no hand-made conflict edits.

If either fails, a `check`-mode reviewer runs on the merge commit before any dependent task
starts. In a parallel run every merge after the first fails condition 1, so it is always
re-checked. That is a deliberate policy — two individually correct changes can break
together — not something a smaller change scope could waive.

**Approvals.** Every approval is recorded in `state.md` with the path and commit of the
file approved. `PRD.md` is approved in the discover session, which is a single session and
records it itself; `PLAN.md` and later approvals are given in the orchestrator's session and
recorded by the orchestrator. No agent writes `Approved:` into `PRD.md` or
`PLAN.md`. Before acting on an approval, the orchestrator confirms the file is unchanged since that commit; if it changed, or no record
exists, it stops and asks.

**Rework is always a fresh builder.** A builder that received `changes required` does not
continue its old session. A `DONE` return already guarantees a clean worktree (§5), so the
new builder starts from the committed HEAD of `task/<id>`. It gets the task, `BASE`, that
HEAD, and the findings from `state.md` — not the previous builder's reasoning — and its first
step is to read `BASE..HEAD` against the findings.

**After STRUCK_OUT or BLOCKED.** The attempts stay in `state.md` under the task and cause,
and the count does not reset with a new session. The worktree is kept. If it has uncommitted
changes, the human decides whether they are committed or discarded; no agent discards them.
A builder is respawned only with the human's answer or new direction recorded in `state.md`,
and receives the earlier attempts.

**Checks have no outside effects, by environment, not by instruction.** The machine agents
run on holds no production credentials at all: production secrets live only in the hosting
platform's and CI's secret stores. Checks run with test configuration and test credentials,
so a mistaken command fails instead of reaching production, real user data, or anything that
costs money or sends messages. Absence cannot be proven by a check the agent runs; the human
confirms it once when setting up the machine, and that confirmation is part of the
precondition at the top of this document.

A check that needs a live external service is listed separately in `PLAN.md`, uses a test or
sandbox account, and runs only with the human's approval for that run. It may read from and
write to that test account only. A check that can cost money or send a message to a real
person is never run by an agent; the human runs it. A live check is never repeated
automatically on resume; the human decides.

### States and transitions

**Persistence.** The orchestrator's checkout always stays on the branch tasks merge into.
At every transition it writes `state.md` and commits it there before acting, so a session
that stops mid-step leaves a committed record of the step it was in. Task branches start
from that branch and never change `state.md`, so a merge always keeps the target branch's
version; there is nothing to reconcile. The orchestrator is the only writer even when tasks
run in parallel — each task has its own section in the one file.

**The plan** has its own status in `state.md`, separate from the tasks:

| From | To | When | Human |
|---|---|---|---|
| — | `drafting` | Planner spawned; recorded first | — |
| `drafting` | `awaiting approval` | Planner `DONE`, verified | — |
| `drafting` | `blocked` | Planner `BLOCKED`, or a malformed or unverifiable return | — |
| `blocked` | `drafting` | The human's answer is recorded; planner respawned | Yes |
| `awaiting approval` | `approved` | The human approves; every task in `PLAN.md` becomes `planned` | Yes |
| `awaiting approval` | `drafting` | The human asks for changes | Yes |
| `approved` | `drafting` | A task needs amending (task `blocked` → `planned` below) | Yes |

**Each task:**

| From | To | When | Human |
|---|---|---|---|
| `planned` | `spawned` | The plan is `approved`; worktree created and verified | — |
| `spawned` | `built` | Builder `DONE`, verified against git | — |
| `spawned` | `blocked` | Builder `BLOCKED`, or a malformed or unverifiable builder return | — |
| `spawned` | `struck out` | Builder `STRUCK_OUT` | — |
| `blocked`, `struck out` | `spawned` | The human's answer is recorded | Yes |
| `blocked` | `planned` | The human decides the task must change; it is `planned` again once the amended plan is `approved` | Yes |
| `built` | `reviewing` | Review worktree created | — |
| `reviewing` | `approved` / `changes required` / `unverified` | Reviewer verdict, verified | — |
| `changes required` | `spawned` | Fresh builder for rework | — |
| `unverified` | `reviewing` or `blocked` | The human decides: review again, or treat as blocked | Yes |
| `approved` | `merged` | The human merges | Yes |
| `merged` | `done` | Merge check: verdict stands, or the merge-commit review approves | — |
| `merged` | `unverified` | The merge-commit review returns `unverified` | — |
| `merged` | `integration failed` | The merge-commit review returns `changes required` | — |
| `integration failed` | `done` | A fix task added by the human is merged, and the check-mode review of that merge commit approves | Yes |

No other transition exists. Nothing returns to `approved` except through a new review, and
nothing is reverted automatically: undoing a merge is the human's action.

**Resuming.** A new orchestrator session compares `state.md` with git before spawning
anything. For a task left `spawned` or `reviewing`, it inspects both the branch and the
worktree. No commits past the last recorded one and a clean worktree: the step is redone.
Anything else — new commits, or uncommitted changes — goes to the human, who decides what is
kept. No agent resets, discards, or force-removes to make resuming possible.

---

## 4. File ownership

One writer per file at any moment. This is what makes parallel builders mergeable.

| File | Writer | Notes |
|---|---|---|
| `PRD.md` | `discover` session | Committed when approved |
| `PLAN.md` | `planner` | Each task names: its requirement, what blocks it, file scope, seams under test, done condition, `parallel: yes/no`. The planner has no `Bash`; the orchestrator commits the file once approved |
| Code, tests | `builder`, on `task/<id>` only | |
| `DECISIONS.md`, `README.md`, `.env.example` | `builder`, in the commit that caused the change | |
| `state.md` | `orchestrate`; when no orchestrator is running, the single session that is — including the `discover` session recording the PRD approval | Committed only on the branch tasks merge into, at every transition. Task branches never change it |
| `known-issues.md` | `orchestrate`, after the human decides to defer | Same branch as `state.md`. A subagent proposes, never writes |
| Review verdict | Nobody writes a file | The orchestrator records it in `state.md`; the approved verdict line goes into the merge commit message the human makes |

**A file outside scope.** A builder that needs to change a file outside its task's scope —
shared test setup, project metadata — returns `BLOCKED` naming the file. The orchestrator
does not widen scope; the plan is amended. Approving `parallel: yes` does not grant either
task the shared file.

**Seams are agreed at Plan.** A builder cannot ask the human which interface to test, so the
planner proposes the seams for each task in `PLAN.md` and the human approves them with the
plan. A builder that finds a seam unworkable returns `BLOCKED` with the seam it proposes
instead and why; the planner amends that task and the human approves the amendment.

---

## 5. Spawn prompt and return message

**Spawn prompt** (orchestrator → subagent) contains only: the task text from `PLAN.md`, the
requirement it traces to, the file scope, the absolute worktree path, the check commands,
and — for rework — the reviewer's findings. For the reviewer: the task text, the base and
head commits, the mode, and the check commands. Never a summary of earlier sessions.

**Return message** (subagent → orchestrator). One schema per role; nothing else in the
message.

`planner`
```
STATUS:   DONE | BLOCKED
NEEDS:    <the decision required>                       required when BLOCKED
```

`builder`
```
STATUS:   DONE | BLOCKED | STRUCK_OUT
TASK:     <PLAN.md task id>
BASE:     <full SHA the branch started from>
COMMIT:   <full SHA of task/<id> HEAD, or none>
BASELINE: <checks failing at BASE before any change, or none>
CHECKS:   <command> → exit <code> | n/a | not run (<reason>)   one line per configured check
NOT REACHED: <paths the checks did not cover, or none>
NEEDS:    <the decision required>                       required when BLOCKED
ATTEMPTS: <cause / change / observation>                required when STRUCK_OUT, one per attempt
```

`reviewer`
```
VERDICT:  approved | changes required | unverified
TASK:     <PLAN.md task id>
COMMIT:   <SHA at start> <SHA at end>
TREE:     clean | dirty                                 review worktree at the end
CHECKS:   <command> → exit <code> | n/a | not run (<reason>)
FINDINGS: <severity> <file:line> <what, how reproduced> <smallest fix>, or none
```

**Malformed or unverifiable returns are never filled in.** A builder or planner message
missing a required field is `BLOCKED`. A reviewer message missing a field is `unverified`.

**The orchestrator checks every return against git before recording it.** Builder: `COMMIT`
exists and is the HEAD of `task/<id>`, `BASE` matches the recorded base, and on `DONE` the
task worktree is clean. Reviewer: both
commits equal the one it was given and `TREE` is clean. Planner: `PLAN.md` is the only
changed file in the checkout. After a reviewer or planner returns, the main checkout and
every task worktree have the HEAD and status they had before the spawn. Any mismatch makes
the return `unverified` (reviewer) or `BLOCKED` (others) and goes to the human.

This check detects changes to files inside the repository and its worktrees. It does not see
files outside the repository or effects on external systems; those are controlled by the
check-environment rule in §3, not detected.

---

## 6. Agent definition files

| Item | Decision |
|---|---|
| Location in this repository | `agents/<name>.md` |
| Location in a project | Copied to `.claude/agents/<name>.md` and committed |
| Language | English |
| Frontmatter | `name`, `description`, `tools` — nothing else |
| Length | SHOULD stay under 120 lines. Every spawn pays for the whole file. A safety rule is never dropped to meet it |
| Content | The role's §8 behaviour, §5 return schema, and inputs. The reasons in §2 and the rejected patterns stay in this document |

Body sections, in this order: **Role** (one paragraph) / **Inputs** / **Outputs** /
**MUST NOT** / **Procedure** (role-specific steps; shared procedures by skill name only) /
**Return** (the §5 schema).

| Choice | Status |
|---|---|
| Frontmatter limited to `name`, `description`, `tools` | Default |
| A CLI-specific copy generated from the canonical file, when a second CLI is adopted | Alternative |
| `model` in a definition | Prohibited — pins a vendor and a price into a shared file; agents inherit the session model |
| `skills:` preloading, `memory:`, `color`, `maxTurns` | Prohibited — tool-specific, and preloading is how plugin bundles occupy context |
| `isolation: worktree` | Prohibited — the orchestrator creates worktrees with `git worktree`, which works under any CLI and makes the path explicit |
| Restating `GLOBAL_RULES.md` or archetype rules in a definition | Prohibited — they arrive via `CLAUDE.md`; a copy drifts |
| Citing section numbers of other documents | Prohibited — section numbers change in step 3 |

**What `tools` enforces and what it does not.** `reviewer` gets no `Edit` or `Write`, but it
needs `Bash` to run checks, and `Bash` can write anywhere. `planner` gets `Write` for
`PLAN.md` and no `Edit` or `Bash`, but `Write` takes any path. Neither is read-only by
configuration. What holds instead: the reviewer works in a disposable worktree that is never
merged, and the orchestrator's after-return check in §5 detects repository changes outside
it and any planner write other than `PLAN.md`, within the limits stated there.

This is a single-CLI design kept convertible, not a proven vendor-independent one. Adopting
a second CLI requires checking that its tool restriction and isolation give the same
guarantees, not only converting the file format.

---

## 7. Worktrees

- The orchestrator creates every worktree: `task/<id>` branch for a builder, detached at
  the commit under review for a reviewer. Paths are absolute and outside the main checkout.
- Before creating one, it checks that the branch and path do not exist. If they do and
  `state.md` records them, the resume rule in §3 applies; if `state.md` does not, the human
  decides.
- After creating one, it confirms the path, the branch, and that HEAD is the recorded base
  commit.
- A builder works only in its worktree. A reviewer never enters the builder's worktree.
- The orchestrator removes a worktree it created only when `git status` there is clean. It
  never force-removes, resets, or deletes a branch to clear a failure; it records the state
  and hands it to the human.
- Procedure: skill `orchestrate`.

---

## 8. Behaviour each definition must carry

What makes each role work, stated as behaviour. Definitions and skills are written from this
section, never by editing someone else's text.

**planner**
- Writes no code, and proposes no plan content before it has read `PRD.md` in full.
- Slices work vertically: each task delivers one thin end-to-end behaviour that can be
  verified on its own, not one layer of many features.
- Each task states what blocks it, the files it may touch, the seams its tests use, and a
  done condition a fresh session can check.
- Asks nothing it can look up in the code. A question only the human can answer goes into
  its return as `BLOCKED`.

**builder**
- Baseline, once, before the first change: runs the configured checks at `BASE` and reports
  what already fails, so failures it did not cause are known.
- Then, for a behaviour change: write one failing test, the least code that passes it,
  repeat. Never write all the new tests before any code.
- Tests go through the agreed seam and assert behaviour, never internals. An expected value
  comes from the requirement or a worked example, never recomputed the way the code does.
- No coverage target.
- Refactoring changes no behaviour, and afterwards every check that passed before it still
  passes; checks that already failed at `BASE` stay recorded, not counted against it. It is
  never mixed into the commit of a behaviour change.
- Returns `DONE` only with every change committed and the worktree clean.
- A bug starts with a test that fails for the stated reason; when the cause is unknown, it
  follows `diagnose`.

**reviewer**
- Its job is to break the claim, not to confirm it. The builder's account is one of the
  things under test, and the reviewer never receives it.
- Reproduces the central claim by running it — a guard is verified by making it fire. Reading
  alone never approves runtime behaviour.
- Records the commit at start and end; if it moved, the verdict is `unverified`. Evidence
  produced at another commit cannot support `approved`.
- Never edits, formats, or fixes, inside or outside the reviewed tree; scratch work goes to a
  temporary directory. `git status` is clean before it returns.
- An error, timeout, missing tool, or unrun central check is `unverified`, never `approved`.
- Severity: **critical** — data loss, a security breach, or a wrong result reaching users with
  no workaround; **high** — a requirement of the task not met, or a defect a user hits in
  normal use; **medium** — a defect only in an edge case; **low** — anything else.
- Any critical or high finding is `changes required`. With only medium or low findings the
  verdict is `approved` and the findings are listed; the orchestrator records them and the
  human decides at merge whether each becomes a new task or goes to `known-issues.md`.
- `check` mode runs the checks the project configures, in this order where present — type
  check, lint, build, test — and reads the diff against the task. A check the project does
  not configure is `n/a`; one that could not run is `not run` with the reason.
- `adversarial` mode adds reproduction and counter-example search in the order security,
  data integrity, correctness, performance.

**discover**
- One question at a time; facts it can look up are not asked.
- Ends only when every open question is answered. Unanswered becomes out of scope, written
  as such, never deferred silently.

**diagnose**
- Reproduce, narrow, form one hypothesis, test it. No fix before a failing test shows the
  cause.
- Escalation goes to the human after three failed attempts at one cause, never to another
  agent with wider authority.

**handoff**
- Writes `state.md` with the next action first and the history after it.
- Facts come from `git` and the files, not from what the session remembers.
- Writes a minimal valid `state.md` before anything else, so a session that runs out of room
  still leaves one.

### Rejected patterns

| Pattern | Why not |
|---|---|
| Installing a ready-made bundle of agents, skills, or hooks | It occupies context at every session start, and its parts assume each other |
| A definition that preloads skills | The same cost, paid on every spawn |
| A verifier that fixes simple errors itself | It is then the author of what it approves |
| A checkpoint stash in the author's working tree | Touches work the agent did not create |
| An escalation chain of increasingly powerful agents | Three failures mean the model of the problem is wrong; a stronger agent repeats it faster |
| A coverage percentage as a done condition | Met by tests that assert nothing |
| Work tracked in an external issue tracker | Files in the repository are the only interface |
| Agents that message each other | Same reason |
| Compressing a session to continue it | The session closes; `state.md` carries what matters |

---

## 9. Changes this design requires elsewhere

`GLOBAL_RULES.md` (step 3):

- Orchestration: "does not edit files" becomes "edits only `state.md` and `known-issues.md`".
  Tracking state is impossible otherwise.
- Scope: "updating `state.md` is always inside scope" holds only when no orchestrator is
  running; in an orchestrated run, subagents report and the orchestrator writes.
- Phases: every phase approval is recorded in `state.md` with the path and commit of the
  approved file, instead of an `Approved:` line in the file itself.

`CLAUDE.md` template: MUST bring `GLOBAL_RULES.md` and the archetype rule into context,
because that is the only way subagents receive them. The import mechanism is not yet
verified.

Skills to draft (step 7): `orchestrate`, `discover`, `handoff` (including the `state.md`
template), `diagnose`. An agent definition that names a skill is not deployed until that
skill exists.
