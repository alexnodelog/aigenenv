# GLOBAL_RULES.md

Applies to every project, from planning through release. The project's `CLAUDE.md` adds
what is specific to that project; it never relaxes a rule here. On conflict, this document
wins.

---

## 1. Principles

Tie-breakers, in order. When two implementations are both reasonable, the one that better
satisfies the higher principle wins.

1. **Maintainable by one person, years from now**, with no memory of writing it.
2. **Simplicity.** The simplest thing that fully solves the actual problem.
3. **Reproducibility.** A clean clone plus the documented commands produces a working
   environment, every time.
4. **The human owns every line**, however much of it the agent wrote.
5. **Vendor independence.** Anything external is replaceable behind an interface the
   project owns.
6. **Explicit over implicit.** Magic saves keystrokes once and costs comprehension forever.

### What simplicity means here

- Solve the problem in front of you, not the one you imagine having later.
- No abstraction until **three** concrete cases demand it. Two is a coincidence.
- No configuration option without a current caller that needs it.
- No design pattern applied because it is a pattern. Name the problem it solves first.
- Prefer deleting code over adding code.
- Dependency injection containers, event buses, plugin systems, and microservices are
  PROHIBITED unless the project cannot work without them and that is recorded in
  `DECISIONS.md`.

### What AI readability means here

Code is read by an agent with no memory of earlier sessions, holding only the files it was
given.

- Names state what the thing is. `user_repository`, not `ur`, `mgr`, or `helper`.
- One file, one responsibility, named after it.
- The same kind of thing lives in the same kind of place with the same kind of name.
- If A depends on B, that is visible in A's imports or signature — never in a naming
  convention, a global, or an import-time side effect.
- Types on every public function signature.

---

## 2. Phases

Work moves through four phases. Each produces a file. A phase MUST NOT begin before the
previous phase's file exists and has been approved by the human.

| Phase | Produces | Done when |
|---|---|---|
| **Discover** | `PRD.md` | The problem, the users, the scope, and what is explicitly out of scope are written down, and open questions are answered — not deferred |
| **Plan** | `PLAN.md` | Work is broken into tasks that can each be verified independently, with their order and dependencies |
| **Build** | Code + tests | Each task passes §11 Definition of Done |
| **Verify** | Review verdict | An independent reviewer approves (§4.3) |

Phase violations to watch for, all PROHIBITED:

- Writing code during Discover or Plan, including "just a quick prototype"
- Starting Build with open questions still in `PRD.md`
- Expanding scope during Build — new scope returns to Discover
- Declaring Verify complete on the author's own assessment

Discover is mostly asking, not writing. An agent in this phase MUST surface what the human
has not specified rather than filling the gap with a plausible assumption. Requirements
ambiguity, not code quality, is the main cause of wasted work.

---

## 3. Execution Protocol

**Human in the loop.** Irreversible decisions are made by the human, not the agent.

**Explain before executing — for irreversible actions only.** State what will happen and
wait for confirmation. Everything else is reported after the fact. Explaining every action
doubles the token cost and trains the human to approve without reading. Risk tiers:

| Tier | Examples | Gate |
|---|---|---|
| **Blocking** | Deleting files or branches, database migrations, rewriting git history, force push, deploying, revoking or rotating credentials, spending money | Explain, then wait for explicit approval. Never proceed on silence |
| **Notify** | Adding or upgrading a dependency, changing public API shape, changing CI configuration, editing more files than the task named | Proceed, then report in the same turn. The human may revert |
| **Silent** | Everything else | Report at the end of the task |

When unsure of a tier, treat it as the higher one.

**Block, don't warn.** A guard that prints a warning will be ignored. Anything that must
not happen is prevented — by a hook, a failing check, or a stop — not advised against.

**Incremental execution.** One verifiable unit at a time. A change that cannot be verified
before the next one begins is too large.

**Inspect the environment before acting on it.** Check what is installed, which version,
which path, which permissions. Never assume a tool, file, or directory exists.

**Speak to the operator, not to a colleague.** When using a term the operator may not
know, define it in a clause. When an action can be undone, say how. When it cannot, say so
before doing it.

---

## 4. Agent Protocol

These rules exist because an unsupervised agent fails in specific, predictable ways.

### 4.1 Scope

Every task states the files or directories the agent may modify. The agent MUST NOT touch
anything outside that scope, and MUST stop and report rather than widen it.

An agent MUST NOT, unasked:

- Refactor or rename anything
- Add a dependency
- Fix an unrelated problem it noticed
- Delete or rewrite a test to make a build pass

Noticing an unrelated problem is useful; reporting it is correct; fixing it in the same
change is not.

### 4.2 Three strikes

On the **same failure three times**, stop, state what was tried and what happened, and ask.
MUST NOT: keep retrying variations, switch architecture to escape the error, disable the
test or type check or lint rule that is reporting it, or hide the symptom behind a
workaround. Three identical failures mean the model of the problem is wrong.

### 4.3 Verification

A task is not complete until the verification that proves it has been run. "It should work"
is not a result. Report what the tests, the type checker, and the lint pass actually said.
What could not be verified MUST be stated, not omitted.

A behavioural change MUST be verified by a reviewer that did not write it and does not see
the author's reasoning — maker is not checker. The reviewer reproduces the claim rather
than reading it, and returns one of: **approved**, **changes required**, or **unverified**.
Only "approved" ends the loop. An author reviewing their own change confirms their own
assumptions.

### 4.4 No fabricated data

An agent MUST NOT invent data to make something run. PROHIBITED outside test fixtures:
placeholder records, sample rows, hard-coded "example" responses, stub API returns, and
fake metrics. If real data or a real endpoint is unavailable, say so and stop. Fabricated
data that reaches a demo becomes a false belief about what works.

Test fixtures are the exception and MUST live under `tests/`, never in application code.

### 4.5 Context discipline

Context is a budget, spent on the current task.

- A sub-session receives the task, its file scope, and the constraints that apply to it —
  not the conversation that produced it.
- When the current task is done and verified, the sub-session closes. State goes to a file
  (§4.6), not to a longer conversation.
- Read the files the task needs. Scanning the repository "for context" is PROHIBITED
  without a stated reason.
- Summarizing work back into the conversation instead of into a file is PROHIBITED — the
  conversation is discarded, the file is not.

### 4.6 Handoff between sessions

Agents communicate through files, never by carrying conversation context forward. This is
what makes closing a session safe.

| File | Holds |
|---|---|
| `PRD.md` | What is being built and why. Phase 1 output |
| `PLAN.md` | Task breakdown, order, dependencies. Phase 2 output |
| `state.md` | Where the work stands now, what is next, what was just verified |
| `known-issues.md` | Problems found and deliberately deferred, with the reason |

Before a session ends, `state.md` MUST reflect reality. A session that closes without that
destroys its own context.

### 4.7 Session start

At the start of every session, before acting: read `CLAUDE.md`, `state.md`, and the
current task's entry in `PLAN.md`. Confirm the repository state matches what `state.md`
claims — branch, uncommitted changes, whether the last task actually landed. When they
disagree, reconcile with the human before doing anything else. A session that begins on a
stale assumption produces work that has to be thrown away.

### 4.8 Parallel agents

- Each agent gets its own git worktree. Two agents MUST NOT share a working directory.
- Each agent gets a non-overlapping file scope.
- Agents MUST NOT merge each other's branches. The human merges.
- One agent per concern. Splitting "API" and "UI" is fine; splitting one function is not.

### 4.9 The orchestrator does not write code

The orchestrating session routes work and tracks state. Once it starts editing files its
context grows without bound and the point of sub-sessions is lost.

---

## 5. Traceability

Every task in `PLAN.md` traces to a requirement in `PRD.md`. Every commit names the task it
implements. Every behavioural change has a test that would fail without it.

- Work with no requirement behind it MUST NOT be started — it is scope creep, and it goes
  back to Discover.
- A requirement with no task MUST NOT be considered delivered.
- Commit messages reference the task identifier. "Fixed stuff" is not a commit message.

For changes an agent made, the record MUST show what changed, which task it served, and
what verified it. When an agent's change is later found wrong, this is what makes the cause
findable instead of guessable.

---

## 6. Repository Structure

Every repository contains:

```
README.md          What it is, how to set it up, how to run it
CLAUDE.md          Agent entry point: what to read, project-specific rules
DECISIONS.md       ADR log (§10)
PRD.md PLAN.md state.md known-issues.md    Phase and handoff artifacts
.env.example       Every environment variable the project reads, with safe placeholders
.gitignore         Includes .env
tests/
```

Plus the lockfiles and version pins the project's stack requires.

`README.md` MUST cover: what the project does, prerequisites, setup commands, how to run
it, how to run the tests. It MUST be accurate — a README that no longer matches the setup
commands is worse than none, because it is trusted.

**Filenames MUST NOT contain version numbers.** Not `rules_v2.md`, not `api_final.py`. Git
is the version record. A file needing a version in its name should have been edited
instead of copied.

---

## 7. Architecture

### 7.1 The dependency rule

Code that expresses what the application *does* MUST NOT directly import or reference:

- A database driver or ORM session
- A vector store client
- A model provider SDK
- An HTTP client or network library
- A hosting or cloud provider SDK
- A web framework, GUI toolkit, or UI library
- The filesystem, the clock, or randomness, where the result affects testability

Dependencies point inward: UI → application logic → domain. The domain knows nothing about
what calls it.

### 7.2 Interfaces at the boundary

Every external system is reached through an interface the project defines and owns, with
implementations in an adapter layer. Replacing the external system MUST touch only the
adapter. One implementation per interface is fine — the rule requires the seam, not a
collection of alternatives.

### 7.3 Where logic does not live

Not in route handlers, controllers, UI event handlers, component bodies, templates,
database queries, or migration scripts. Those are delivery mechanisms; they call logic,
they do not contain it.

### 7.4 Design discipline

- **Single responsibility.** A module has one reason to change. When describing what it
  does requires "and", split it.
- **Depend on abstractions**, not concrete implementations, at every boundary crossing.
- **Don't repeat yourself** — but duplication is cheaper than the wrong abstraction. Two
  similar blocks stay duplicated until the third appears and proves the shape.
- **Composition over inheritance.** Inheritance deeper than one level requires a recorded
  reason.

### 7.5 State

Global mutable state is PROHIBITED. Configuration is read once at startup and passed
explicitly. Singletons are PROHIBITED except for genuinely process-wide resources — a
connection pool, a logger — and those are injected, not reached for.

---

## 8. Code Quality

| Rule | Threshold | On exceeding |
|---|---|---|
| File length | 500 lines | Split by responsibility, not by line count |
| Function length | 50 lines | It is doing more than one thing |
| Nesting depth | 3 | Use early returns and guard clauses |
| Parameters | 5 | Group related ones into a type |

These trigger a review, not a build failure. Crossing one means asking whether the
structure is right.

- Public functions MUST have type annotations on parameters and return value.
- `any` in TypeScript and untyped `dict` passing in Python are PROHIBITED as a substitute
  for defining a type.
- Commented-out code MUST NOT be committed. Dead code MUST be deleted, not kept "just in
  case". Git remembers.
- Magic numbers and strings MUST be named constants.
- Comments explain **why**, never **what**.

### Dependencies

Before adding one: does the standard library do this? Does an existing dependency? Is the
problem big enough to justify it?

- Lockfiles MUST be committed.
- A dependency added for a single function SHOULD be replaced by that function.
- An unmaintained dependency MUST NOT be adopted. Check the last release date.
- Dependency updates are checked for known vulnerabilities before merging, and a dependency
  with an unpatched advisory MUST NOT ship.

---

## 9. Testing

- A bug fix MUST start with a failing test that reproduces the bug, confirmed to fail for
  the stated reason. A bug fixed without one is unobserved, not fixed.
- A new feature SHOULD start with its tests. Where impractical, tests ship in the same
  commit.
- The default test run MUST work offline. Tests needing network access are integration
  tests, MUST be marked, and MUST be excluded from the default run.
- Tests MUST be deterministic. An intermittently failing test is fixed or deleted, never
  retried into passing.
- Tests MUST NOT depend on execution order or each other's state.
- A test MUST fail for exactly one reason.
- Domain logic MUST have meaningful unit coverage. A coverage percentage is not a goal and
  MUST NOT be met with tests that assert nothing.
- Fixing a failing test by changing the assertion is PROHIBITED unless the assertion was
  provably wrong, and that reasoning goes in the commit message.

---

## 10. Errors, Logging, Security

**Errors.** Bare `except:` / `catch {}` that swallows an exception is PROHIBITED. Catch
only what can actually be handled; everything else propagates. Fail fast and loudly in
development — never degrade silently. Error messages state what failed and what to do about
it. User-facing errors MUST NOT expose stack traces, file paths, or internal identifiers.

**Logging.** Structured, with levels used as intended. Secrets, credentials, tokens, and
personal data MUST NOT be logged at any level. `print` / `console.log` MUST NOT be
committed as logging.

**Security.**

- Secrets MUST NOT appear in source, commits, logs, or error messages. They come from the
  environment, and `.env.example` documents every one.
- `.env` MUST be in `.gitignore` before the first commit.
- All external input — user input, API responses, file contents, environment variables —
  MUST be validated at the boundary before reaching business logic.
- SQL MUST be parameterized. String-built queries are PROHIBITED.
- A committed secret MUST be revoked, not just removed from the next commit.
- Least privilege: every credential, token, and service account gets the narrowest scope
  that works. An agent MUST NOT be given write access it does not need for the current
  task.

---

## 11. Git, Release, Decisions

- Conventional Commits. One logical change per commit — a message needing "and" is two
  commits.
- `main` is always in a working state. Force-pushing to `main` is PROHIBITED.
- `--no-verify` is PROHIBITED. If a hook is wrong, fix the hook.
- Release versions follow Semantic Versioning, tagged in Git. Release artifacts are
  produced by CI, never from a developer's machine.
- CI MUST verify, before any release: the full test suite passes, type check passes, lint
  passes, the build succeeds from a clean checkout, and no secret is present in the tree.
  A release that skipped a check is rolled back, not patched forward.

`DECISIONS.md` gets an entry when a choice deviates from the project's stated stack,
selects a documented alternative, will be non-obvious later, or reverses an earlier
decision. Entries are append-only; a reversal is a new entry, and the superseded one stays
with its status changed. The same applies to a rule in this document that was deliberately
broken: the override is recorded with its reason, or it did not happen.

```markdown
## ADR-0001: <decision in one line>
**Date:** YYYY-MM-DD
**Status:** Accepted | Superseded by ADR-00XX

**Context.** What forced a choice.
**Decision.** What was chosen.
**Consequences.** What this costs and rules out.
**Alternatives.** What was rejected, and why.
```

---

## 12. Definition of Done

- [ ] Does what the task asked, and nothing it did not ask
- [ ] Traces to a requirement in `PRD.md`
- [ ] Tests exist; the full default suite passes offline
- [ ] Type check, lint, and format pass without `--no-verify`
- [ ] Verified by a reviewer that did not write it
- [ ] No secrets, fabricated data, commented-out code, dead code, or debug prints
- [ ] Public functions typed
- [ ] `README.md` updated if setup or usage changed
- [ ] `DECISIONS.md` updated if a decision needs an ADR
- [ ] `.env.example` updated if a variable was added
- [ ] `state.md` reflects reality
- [ ] Committed with a Conventional Commit message naming the task

"It works on my machine" is not on this list and never satisfies it.
