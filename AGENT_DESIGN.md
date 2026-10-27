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
     ├─ per task, in PLAN.md order (parallel only where §5.9 allows):
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
| `PLAN.md` | `planner` | Each task names: its requirement, file scope, seams under test, done condition. The planner has no `Bash`; the orchestrator commits the file once approved |
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
**Return** (the §5 shape) / **Source** (provenance line, see §8).

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

## 8. Sources

Repositories inspected at the commits below. Both are MIT; a definition or skill that copies
text from either MUST keep the copyright line and name the source and commit in its
**Source** section.

| Source | Commit |
|---|---|
| `sangrokjung/claude-forge` | `34d881dc9bdc669aadc3a1e8147a4bd5467ecbe3` (2026-09-03) |
| `mattpocock/skills` | `5beb35b040e3de97e54fa9e386d7e42d15c295c2` (2026-10-19) |

| Ours | Taken from | Rewritten because |
|---|---|---|
| `planner` | Forge `planner` (no code before approval); Matt `to-tickets` (vertical slices, blocking edges) | Forge's preloads Superpowers skills and returns the plan as a message; Matt's publishes to an issue tracker. Ours writes `PLAN.md` and adds seams per task |
| `builder` | Forge `tdd-guide`; Matt `tdd` (seams, anti-patterns, one slice per cycle) | Forge's demands 80% coverage, which the testing rules forbid as a goal, and preloads Superpowers; Matt's confirms seams with the user mid-build, which a subagent cannot do |
| `reviewer` | Forge `adversarial-reviewer` (near-verbatim invariants and protocol); Forge `verify-agent` (check order, recording the commit) | Verdict words changed to the ones `GLOBAL_RULES.md` uses; runs in its own worktree instead of the author's tree. `verify-agent`'s auto-fix and `git stash` checkpoint are dropped — a verifier that edits is no longer independent, and it stashes in the author's tree |
| `discover` | Matt `grill-me` / `grilling`; Matt `to-spec` | Output fixed to `PRD.md`; open questions must be answered, not listed |
| `diagnose` | Matt `diagnosing-bugs`; Forge `systematic-debugger` | Forge's escalation chain (`rca-debugger`, `escalation-fixer`) contradicts three strikes: escalation goes to the human |
| `handoff` | Forge `relay` (skeleton first, facts from git not memory, next task first); Matt `handoff` | No `/compact`: the session closes. Target is `state.md`, not a temp file |
| `orchestrate` | — | No suitable source inspected |

### Not adopted

| Item | Reason |
|---|---|
| Any plugin or full install (Forge, Superpowers, Matt's plugin) | Context occupied at session start |
| Forge's other agents (`architect`, `code-reviewer`, `security-reviewer`, `rca-debugger`, `escalation-fixer`, …) | No role in the flow that the three subagents do not already cover |
| Matt `triage` | Moves issues through an issue tracker; this flow has none |
| Matt `improve-codebase-architecture` | Not a phase. Reconsider as a skill if a concrete need appears |
| Matt `git-guardrails-claude-code` | A hook, not an agent. Belongs with the guards in step 3 |
| Matt `zoom-out` | Removed upstream |
| Agent teams / agents messaging each other | Bypasses files as the only interface |

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
