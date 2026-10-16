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

## 2. Execution Protocol

**Human in the loop.** Irreversible decisions are made by the human, not the agent.

**Explain before executing — for irreversible actions only.** State what will happen and
wait for confirmation before: deleting files, running migrations, deploying, rewriting git
history, adding a dependency, or changing anything outside the stated scope. Everything
else is reported after the fact. Explaining every action doubles the token cost and trains
the human to approve without reading.

**Incremental execution.** One verifiable unit at a time. A change that cannot be verified
before the next one begins is too large.

**Inspect the environment before acting on it.** Check what is installed, which version,
which path, which permissions. Never assume a tool, a file, or a directory exists. A
command that fails because of an unchecked assumption wastes a cycle and can do damage.

**Speak to the operator, not to a colleague.** When using a term the operator may not
know, define it in a clause. When an action can be undone, say how. When it cannot, say so
before doing it.

---

## 3. Agent Protocol

These rules exist because an unsupervised agent fails in specific, predictable ways.

### 3.1 Scope

Every task states the files or directories the agent may modify. The agent MUST NOT touch
anything outside that scope, and MUST stop and report rather than widen it.

An agent MUST NOT, unasked:

- Refactor or rename anything
- Add a dependency
- Fix an unrelated problem it noticed
- Delete or rewrite a test to make a build pass

Noticing an unrelated problem is useful; reporting it is correct; fixing it in the same
change is not.

### 3.2 Three strikes

On the **same failure three times**, stop, state what was tried and what happened, and ask.
MUST NOT: keep retrying variations, switch architecture to escape the error, disable the
test or type check or lint rule that is reporting it, or hide the symptom behind a
workaround. Three identical failures mean the model of the problem is wrong.

### 3.3 Verification

A task is not complete until the verification that proves it has been run. "It should work"
is not a result. Report what the tests, the type checker, and the lint pass actually said.
What could not be verified MUST be stated, not omitted.

A behavioural change SHOULD be verified by a reviewer that did not write it and does not
see the author's reasoning. An author reviewing their own change confirms their own
assumptions.

### 3.4 Handoff between sessions

Agents communicate through files, never by carrying conversation context forward. This is
what makes closing a session safe.

| File | Holds |
|---|---|
| `PRD.md` | What is being built and why. Written before design |
| `PLAN.md` | The task breakdown. Written before implementation |
| `state.md` | Where the work currently stands, and what is next |
| `known-issues.md` | Problems found and deliberately deferred |

Before a session ends, `state.md` MUST reflect reality. A session that closes without that
destroys its own context.

### 3.5 Parallel agents

- Each agent gets its own git worktree. Two agents MUST NOT share a working directory.
- Each agent gets a non-overlapping file scope.
- Agents MUST NOT merge each other's branches. The human merges.
- One agent per concern. Splitting "API" and "UI" is fine; splitting one function is not.

### 3.6 The orchestrator does not write code

The orchestrating session routes work and tracks state. Once it starts editing files its
context grows without bound and the point of sub-sessions is lost.

---

## 4. Repository Structure

Every repository contains:

```
README.md          What it is, how to set it up, how to run it
CLAUDE.md          Agent entry point: what to read, project-specific rules
DECISIONS.md       ADR log (§9)
PRD.md PLAN.md state.md known-issues.md    Handoff artifacts (§3.4)
.env.example       Every environment variable the project reads, with safe placeholders
.gitignore         Includes .env
tests/
```

Plus the lockfiles and version pins the project's stack requires.

**Filenames MUST NOT contain version numbers.** Not `rules_v2.md`, not `api_final.py`. Git
is the version record. A file needing a version in its name should have been edited
instead of copied.

---

## 5. Architecture

### 5.1 The dependency rule

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

### 5.2 Interfaces at the boundary

Every external system is reached through an interface the project defines and owns, with
implementations in an adapter layer. Replacing the external system MUST touch only the
adapter. One implementation per interface is fine — the rule requires the seam, not a
collection of alternatives.

### 5.3 Where logic does not live

Not in route handlers, controllers, UI event handlers, component bodies, templates,
database queries, or migration scripts. Those are delivery mechanisms; they call logic,
they do not contain it.

### 5.4 State

Global mutable state is PROHIBITED. Configuration is read once at startup and passed
explicitly. Singletons are PROHIBITED except for genuinely process-wide resources — a
connection pool, a logger — and those are injected, not reached for.

---

## 6. Code Quality

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

---

## 7. Testing

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

## 8. Errors, Logging, Security

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

---

## 9. Git and Decisions

- Conventional Commits. One logical change per commit — a message needing "and" is two
  commits.
- `main` is always in a working state. Force-pushing to `main` is PROHIBITED.
- `--no-verify` is PROHIBITED. If a hook is wrong, fix the hook.
- Release versions follow Semantic Versioning, tagged in Git. Release artifacts are
  produced by CI, never from a developer's machine.

`DECISIONS.md` gets an entry when a choice deviates from the project's stated stack,
selects a documented alternative, will be non-obvious later, or reverses an earlier
decision. Entries are append-only; a reversal is a new entry, and the superseded one stays
with its status changed.

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

## 10. Definition of Done

- [ ] Does what was asked, and nothing that was not asked
- [ ] Tests exist; the full default suite passes offline
- [ ] Type check, lint, and format pass without `--no-verify`
- [ ] No secrets, commented-out code, dead code, or debug prints
- [ ] Public functions typed
- [ ] `README.md` updated if setup or usage changed
- [ ] `DECISIONS.md` updated if a decision needs an ADR
- [ ] `.env.example` updated if a variable was added
- [ ] `state.md` reflects reality
- [ ] Committed with a Conventional Commit message

"It works on my machine" is not on this list and never satisfies it.
