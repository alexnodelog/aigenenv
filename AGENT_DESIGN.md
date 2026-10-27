# AGENT_DESIGN.md

Which agents exist, where each one's work starts and stops, and how they hand work to each
other. This document is **not deployed**. Like `TECH_STACK.md`, it is the source from which
deployed files are derived: `agents/*.md` (agent definitions) and the orchestration skills.
Edit here first, then regenerate the derived files. Deployed files MUST NOT reference this
document.

Precedence is unchanged: `GLOBAL_RULES.md → rules/<project>.md → CLAUDE.md → current task`.
An agent definition adds role-specific behaviour; it never relaxes a rule.

---

## 1. Facts this design depends on

Checked 2026-10-26. Re-check before changing a decision that rests on one.

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
| `discover` | Skill, in an interactive session with the human | Discover | The request, existing `PRD.md`, code only to answer a factual question | `PRD.md` | Writes code or `PLAN.md`; leaves an open question in `PRD.md` |
| `orchestrate` | Skill, in the main session | Plan → Verify | `PRD.md`, `PLAN.md`, `state.md`, subagent return messages | `state.md`, `known-issues.md` | Edits code, tests, `PRD.md`, `PLAN.md`; forwards a builder's message to the reviewer; merges |
| `planner` | Subagent | Plan | `PRD.md`, the code it needs to size tasks | `PLAN.md` | Writes code; adds a task with no requirement behind it |
| `builder` | Subagent, in a task worktree | Build | One `PLAN.md` task, its requirement, the code in scope | Code, tests, `DECISIONS.md`, `README.md`, `.env.example` — on its task branch | Touches `state.md`; widens scope; starts another task |
| `reviewer` | Subagent, in a detached worktree at the commit under review | Verify | The task text, the diff range, the check commands, the repository at that commit | Nothing in the repository | Reads `state.md` or `known-issues.md`; fixes what it finds |

Supporting skills, loaded only when used: `handoff` (closing a session into `state.md`),
`diagnose` (a bug whose cause is unknown; used by `builder` and by single sessions).

**Small tasks use none of this.** One session does the task and the human reads the diff.
The orchestrator exists only when Standard work is split across sessions.

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
     │     create worktree + branch task/<id>
     │     spawn builder ──▶ commits on task/<id>, returns status
     │     record status in state.md
     │     create detached worktree at the returned commit
     │     spawn reviewer (mode by task) ──▶ returns verdict
     │     record verdict in state.md
     │       approved            → human merges task/<id>
     │       changes required    → findings into state.md, spawn a fresh builder
     │       unverified          → human decides
     │
     └─ BLOCKED / STRUCK_OUT from any subagent → ask the human, record the answer, respawn
```

**One task, one human checkpoint.** Every task ends with a reviewer verdict and the human
merging its branch. The next task that depends on it starts only after that merge.

**Sequential or parallel.** Tasks run one at a time by default. Two tasks MAY run at the same
time only when `PLAN.md` marks both `parallel: yes`, which the planner may do only when
neither blocks the other and their file scopes do not overlap. The human approves this with
the plan; the orchestrator never decides it on its own. Both modes use the same procedure —
a worktree and a `task/<id>` branch per task — so switching between them changes nothing
else.

**Reviewer mode.** `adversarial` when the task is in the Always-Standard list or changes a
public interface; `check` for every other Standard task. The orchestrator states the mode
in the spawn prompt; the reviewer does not choose it.

**Approvals in an orchestrated run.** The human approves in the orchestrator's session, and
the orchestrator records it in `state.md` with the commit of the file approved. The
orchestrator does not write `Approved:` into `PRD.md` or `PLAN.md`.

**Rework is always a fresh builder.** A builder that received `changes required` does not
continue its old session. The new builder gets the task, the findings from `state.md`, and
the existing worktree — not the previous builder's reasoning.

---

## 4. File ownership

One writer per file at any moment. This is what makes parallel builders mergeable.

| File | Writer | Notes |
|---|---|---|
| `PRD.md` | `discover` session | Committed when approved |
| `PLAN.md` | `planner` | Each task names: its requirement, what blocks it, file scope, seams under test, done condition, `parallel: yes/no`. The planner has no `Bash`; the orchestrator commits the file once approved |
| Code, tests | `builder`, on `task/<id>` only | |
| `DECISIONS.md`, `README.md`, `.env.example` | `builder`, in the commit that caused the change | |
| `state.md` | `orchestrate`; a single session when there is no orchestrator | Committed by the orchestrator on the branch its checkout is on |
| `known-issues.md` | `orchestrate`, after the human decides to defer | Deferral is a human decision, so a subagent proposes and never writes |
| Review verdict | Nobody writes a file | The orchestrator records it in `state.md`; the approved verdict line goes into the merge commit message the human makes |

**Seams are agreed at Plan, not Build.** A builder cannot ask the human which interface to
test. The planner proposes the seams for each task in `PLAN.md`, and the human approves them
with the plan. A builder that finds the agreed seam unworkable returns `BLOCKED`.

---

## 5. Spawn prompt and return message

**Spawn prompt** (orchestrator → subagent) contains only: the task text from `PLAN.md`, the
requirement it traces to, the file scope, the absolute worktree path, the check commands,
and — for rework — the reviewer's findings. For the reviewer: the task text, the base and
head commits, the mode, and the check commands. Never a summary of earlier sessions.

**Return message** (subagent → orchestrator), every field present, nothing else:

```
STATUS:   DONE | BLOCKED | STRUCK_OUT            (reviewer: approved | changes required | unverified)
TASK:     <PLAN.md task id>
COMMIT:   <full SHA>                              (reviewer: the commit examined)
CHECKS:   <check>: passed | failed | not run (<reason>)   — one line each
NOT REACHED: <paths the checks did not cover>
NEEDS:    <the decision required>                 (BLOCKED only)
ATTEMPTS: <cause / change / observation>           (STRUCK_OUT only, one line per attempt)
FINDINGS: <severity> <file:line> <what, how reproduced> <smallest fix>   (reviewer only)
```

The orchestrator writes the relevant fields into `state.md` before doing anything else with
them. A return message that is missing a field is treated as `unverified`/`BLOCKED`, not
filled in.

---

## 6. Agent definition files

| Item | Decision |
|---|---|
| Location in this repository | `agents/<name>.md` |
| Location in a project | Copied to `.claude/agents/<name>.md` and committed |
| Language | English |
| Frontmatter | `name`, `description`, `tools` — nothing else |
| Length | SHOULD stay under 120 lines. Every spawn pays for the whole file |

Body sections, in this order: **Role** (one paragraph) / **Inputs** / **Outputs** /
**MUST NOT** / **Procedure** (role-specific steps; shared procedures by skill name only) /
**Return** (the §5 shape).

| Choice | Status |
|---|---|
| Frontmatter limited to `name`, `description`, `tools` | Default |
| A CLI-specific copy generated from the canonical file, when a second CLI is adopted | Alternative |
| `model` in a definition | Prohibited — pins a vendor and a price into a shared file; agents inherit the session model |
| `skills:` preloading, `memory:`, `color`, `maxTurns` | Prohibited — tool-specific, and preloading is how plugin bundles occupy context |
| `isolation: worktree` | Prohibited — the orchestrator creates worktrees with `git worktree`, which works under any CLI and makes the path explicit |
| Restating `GLOBAL_RULES.md` or archetype rules in a definition | Prohibited — they arrive via `CLAUDE.md`; a copy drifts |
| Citing section numbers of other documents | Prohibited — section numbers change in step 3 |

`tools` is the only enforcement the frontmatter gives, and it is coarse. `reviewer` gets no
`Edit` or `Write`; it needs `Bash` to run checks, so its read-only guarantee comes from the
detached worktree plus a `git status` check before it returns. `planner` gets `Write`
because `PLAN.md` is its output, and no `Edit` or `Bash`; its definition limits `Write` to
that one file.

---

## 7. Worktrees

- The orchestrator creates every worktree: `task/<id>` branch for a builder, detached at
  the commit under review for a reviewer. Paths are absolute and outside the main checkout.
- A builder works only in its worktree. A reviewer never enters the builder's worktree.
- The orchestrator removes a worktree it created only when `git status` there is clean.
  Anything else is reported to the human.
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
- One failing test, the least code that passes it, repeat. Never all tests first.
- Tests go through the agreed seam and assert behaviour, never internals. An expected value
  comes from the requirement or a worked example, never recomputed the way the code does.
- No coverage target. Refactoring is outside the loop.
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
  Any critical or high finding is `changes required`. There is no fourth verdict.
- `check` mode runs the project's checks in order — type check, lint, build, test — and reads
  the diff against the task. `adversarial` mode adds reproduction and counter-example search
  in the order security, data integrity, correctness, performance.

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
- Phases: an approval given in an orchestrated session is recorded in `state.md` with the
  commit of the approved file.

`CLAUDE.md` template: MUST bring `GLOBAL_RULES.md` and the archetype rule into context,
because that is the only way subagents receive them. The import mechanism is not yet
verified.

Skills to draft (step 7): `orchestrate`, `discover`, `handoff` (including the `state.md`
template), `diagnose`.
