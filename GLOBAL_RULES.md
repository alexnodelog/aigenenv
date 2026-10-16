# GLOBAL_RULES.md

**What this is.** The rules that apply to every project in this framework, regardless of
what it is or what it is built with. This document decides *how* to build.
`TECH_STACK.md` decides *what to build with*. Neither repeats the other.

**Precedence.** `GLOBAL_RULES.md` → `TECH_STACK.md` → `rules/<project>.md` → current task.
A lower document MAY narrow a rule set here. It MUST NOT widen one. A conflict between
documents is a defect in the documents, not a judgement call at coding time — stop and fix
the document.

**RFC 2119.** MUST / MUST NOT / SHOULD / SHOULD NOT / MAY carry their RFC 2119 meaning.

**Audience.** Both a human and an AI agent. Every rule here is written to be actionable
without further interpretation. A rule that needs explaining is a rule that needs
rewriting.

---

## 1. Principles

These are the tie-breakers. When two reasonable implementations exist, the one that better
satisfies the higher principle wins.

1. **Maintainable by one person, years from now.** The maintainer is the author, with no
   memory of writing it. Every other principle serves this one.
2. **Simplicity first.** The simplest thing that fully solves the actual problem. Not the
   most clever, not the most general, not the most impressive.
3. **Reproducibility.** A clean clone plus the documented setup commands produces a working
   environment, every time, on every machine.
4. **AI as partner.** The AI writes most of the code. The human owns every line of it.
5. **Vendor independence.** Anything external is replaceable behind an interface the
   project owns.
6. **Explicit over implicit.** Magic saves keystrokes once and costs comprehension forever.

### 1.1 Simplicity, concretely

Simplicity is not a feeling. These are the rules it produces:

- Solve the problem in front of you. Not the one you imagine having later.
- No abstraction until there are **three** concrete cases demanding it. Two is a
  coincidence.
- No configuration option without a current caller that needs it.
- No design pattern applied because it is a pattern. Name the problem it solves first.
- Prefer deleting code over adding code. A feature removed is a feature that cannot break.
- Enterprise scaffolding — dependency injection containers, event buses, plugin systems,
  microservices — is PROHIBITED unless the project cannot work without it and that is
  written down in `DECISIONS.md`.

### 1.2 AI readability, concretely

Code is read by an AI agent with no memory of previous sessions, holding only the files it
was given. That constrains how it must be written:

- Names say what the thing is. `user_repository`, not `ur` or `mgr` or `helper`.
- One file, one responsibility, with a name that states it.
- Structure is consistent across the project. The same kind of thing lives in the same kind
  of place with the same kind of name.
- No implicit coupling. If A depends on B, that dependency is visible in A's imports or
  signature — never in a naming convention, a global, or a side effect at import time.
- Types on every public function signature. An agent that must infer a type will guess
  wrong eventually.

---

## 2. Working with AI Agents

This section exists because an unsupervised agent fails in specific, predictable ways.
These rules are the countermeasures.

### 2.1 Scope discipline

Every task given to an agent MUST state the files or directories it may modify. The agent
MUST NOT modify anything outside that scope. When the task appears to require a change
outside scope, the agent MUST stop and report, not expand the scope on its own.

An agent MUST NOT:

- Refactor code it was not asked to refactor
- Rename things it was not asked to rename
- Add dependencies without being asked
- "Fix" unrelated issues it noticed along the way
- Delete or rewrite tests in order to make a build pass

Noticing an unrelated problem is useful. Reporting it is correct. Fixing it in the same
change is not.

### 2.2 The three-strike rule

When an agent hits the **same failure three times**, it MUST stop, state plainly what it
tried and what happened, and ask for human input. It MUST NOT:

- Keep retrying variations of the same approach
- Switch to a fundamentally different architecture to escape the error
- Disable the test, the type check, or the lint rule that is reporting the failure
- Add a workaround that hides the symptom

Three identical failures mean the agent's model of the problem is wrong. More attempts
produce more damage, not a solution.

### 2.3 Tests before fixes

For a **bug fix**, a failing test that reproduces the bug MUST be written first, and MUST
be confirmed to fail for the stated reason, before any fix is attempted. A bug fixed
without a reproducing test is not fixed; it is unobserved.

For a **new feature**, tests SHOULD be written first. Where that is impractical, tests MUST
accompany the feature in the same commit.

### 2.4 Parallel agents

When more than one agent works on a repository at once:

- Each agent MUST have its own Git worktree. Two agents MUST NOT share a working directory.
- Each agent MUST have a non-overlapping file scope.
- Agents MUST NOT merge each other's branches. The human merges.
- One agent per concern. Splitting "write the API" and "write the UI" across agents is
  fine; splitting one function across agents is not.

These rules are about behaviour, not tools. Any tool satisfying them is acceptable, and
they survive the current tools being replaced.

### 2.5 Verification before claiming

An agent MUST NOT report a task complete without having run the verification that proves it.
"It should work" is not a result. Running the tests, the type checker, and the lint pass —
and reporting what they actually said — is.

When something could not be verified, that MUST be stated explicitly rather than omitted.

---

## 3. Repository Structure

Every project repository MUST contain:

```
README.md          What it is, how to set it up, how to run it
CLAUDE.md          Entry point for the AI agent: what to read, project-specific context
DECISIONS.md       ADR log (see §9)
.env.example       Every environment variable the project reads, with safe placeholders
.gitignore         Includes .env
tests/             Test suite
```

Plus the lockfiles and version pins `TECH_STACK.md` requires for the chosen stack.

**Filenames MUST NOT contain version numbers.** Not `rules_v2.md`, not
`api_final_revised.py`. Git is the version record. A file that needs a version in its name
is a file that should have been edited instead of copied.

Directory layout beyond the above is set per archetype in `rules/<project>.md`.

---

## 4. Architecture

### 4.1 The dependency rule

Business logic MUST NOT depend on infrastructure. Code that expresses what the application
*does* MUST NOT directly import or reference:

- A database driver or ORM session — SQLite, PostgreSQL, D1, SQLAlchemy, Drizzle
- A vector store client — FAISS, Qdrant, pgvector, sqlite-vec
- A model provider SDK — OpenAI, Ollama, Gemini, Anthropic
- An HTTP client or any network library
- A hosting or cloud provider SDK
- A web framework, GUI toolkit, or UI library — FastAPI, Next.js, PySide6, React, Electron
- The filesystem, the clock, or randomness, where the result affects testability

Dependencies point inward: UI → application logic → domain. Never the reverse. The domain
layer knows nothing about what calls it.

### 4.2 Interfaces at the boundary

Every external system is reached through an interface the project defines and owns.
Implementations live in an adapter layer. Replacing the external system MUST touch only the
adapter.

A project MAY ship exactly one implementation per interface. The rule requires the seam,
not a collection of alternatives built in advance.

### 4.3 Where logic lives

Business logic MUST NOT live in:

- Route handlers or controllers
- UI event handlers or component bodies
- Templates
- Database queries or stored procedures
- Migration scripts

Those are delivery mechanisms. They call logic; they do not contain it.

### 4.4 State

Global mutable state is PROHIBITED. Configuration is read once at startup and passed
explicitly. Singletons are PROHIBITED except for genuinely process-wide resources — a
connection pool, a logger — and those are injected, not reached for.

---

## 5. Code Quality

| Rule | Limit | On exceeding |
|---|---|---|
| File length | 500 lines | Split by responsibility, not by line count |
| Function length | 50 lines | The function is doing more than one thing |
| Nesting depth | 3 levels | Use early returns and guard clauses |
| Function parameters | 5 | Group related parameters into a type |

These are thresholds for *review*, not hard failures. Crossing one means stopping to ask
whether the structure is right — and most of the time it is not.

Additionally:

- Every public function MUST have type annotations on parameters and return value.
- `any` in TypeScript and untyped `dict` passing in Python are PROHIBITED as a way of
  avoiding a type definition.
- Commented-out code MUST NOT be committed. Delete it; Git remembers.
- Dead code — unreachable, unused, or unreferenced — MUST be deleted, not left "just in
  case".
- Magic numbers and magic strings MUST be named constants.
- Comments explain **why**, never **what**. A comment restating the code is noise; a
  comment explaining a non-obvious decision is essential.

### 5.1 Dependencies

Before adding a dependency, answer: does the standard library do this? Does an existing
dependency do this? Is the problem large enough to justify a dependency at all?

- Lockfiles MUST be committed.
- A dependency added for a single function SHOULD be replaced by that function.
- An unmaintained dependency MUST NOT be adopted. Check the last release date.

---

## 6. Testing

- The default test run MUST work offline. Tests that require network access are integration
  tests, MUST be marked as such, and MUST be excluded from the default run.
- Tests MUST be deterministic. A test that fails intermittently MUST be fixed or deleted,
  never retried into passing.
- Tests MUST NOT depend on execution order or on each other's state.
- Domain logic MUST have meaningful unit test coverage. Adapters SHOULD be covered by
  integration tests. A global coverage percentage is not a goal in itself and MUST NOT be
  gamed with tests that assert nothing.
- A test MUST fail for exactly one reason. Tests that assert ten things tell you nothing
  when they break.
- Fixing a failing test by changing the assertion is PROHIBITED unless the assertion was
  provably wrong, and that reasoning MUST appear in the commit message.

---

## 7. Errors, Logging, Security

### 7.1 Errors

- Bare `except:` / `catch {}` that swallows an exception is PROHIBITED.
- Catch the specific exception you can actually handle. Everything else propagates.
- Fail fast and loudly in development. Never degrade silently.
- Error messages state what failed and what the caller should do about it.
- User-facing errors MUST NOT expose stack traces, file paths, or internal identifiers.

### 7.2 Logging

- Logging is structured, with levels used as intended: DEBUG for development detail, INFO
  for state changes, WARNING for recoverable problems, ERROR for failures.
- Secrets, credentials, tokens, and personal data MUST NOT be logged at any level.
- `print` / `console.log` MUST NOT be committed as logging.

### 7.3 Security

- Secrets MUST NOT appear in source, in commits, in logs, or in error messages. They come
  from the environment, and `.env.example` documents every one of them.
- `.env` MUST be in `.gitignore` before the first commit, not after.
- All external input — user input, API responses, file contents, environment variables —
  MUST be validated at the boundary before reaching business logic.
- SQL MUST be parameterized. String-built queries are PROHIBITED.
- A committed secret is a rotated secret. Removing it from the next commit is not
  sufficient; it MUST be revoked.

---

## 8. Git and Releases

- Commits follow Conventional Commits.
- One logical change per commit. A commit that needs "and" in its message is two commits.
- `main` MUST always be in a working state.
- Force-pushing to `main` is PROHIBITED.
- `--no-verify` is PROHIBITED. If a pre-commit hook is wrong, fix the hook.
- Branch names describe the work: `feat/user-export`, `fix/login-redirect`.
- Release versions follow Semantic Versioning, tagged in Git.
- Release artifacts MUST be produced by CI, never from a developer's machine.

---

## 9. Decisions

Every project keeps `DECISIONS.md`. An entry is required when a choice:

- Deviates from `TECH_STACK.md`
- Selects an Alternative where a Default exists
- Will be non-obvious to someone reading the code later
- Was reversed — the reversal is a new entry, not an edit to the old one

Format:

```markdown
## ADR-0001: <decision in one line>
**Date:** YYYY-MM-DD
**Status:** Accepted | Superseded by ADR-00XX

**Context.** What situation forced a choice.
**Decision.** What was chosen.
**Consequences.** What this costs, and what it rules out.
**Alternatives considered.** What was rejected, and why.
```

Entries are append-only. A superseded decision stays in the file with its status changed.
The history of what was tried and abandoned is the most valuable part of the file.

---

## 10. Definition of Done

A task is complete only when all of the following are true:

- [ ] It does what was asked, and nothing that was not asked
- [ ] Tests exist, and the full default suite passes offline
- [ ] Type check passes
- [ ] Lint and format pass without `--no-verify`
- [ ] No secrets, no commented-out code, no dead code, no debug prints
- [ ] Public functions are typed
- [ ] `README.md` reflects any change to setup or usage
- [ ] `DECISIONS.md` records any choice requiring an ADR
- [ ] `.env.example` lists any new environment variable
- [ ] Changes are committed with a Conventional Commit message

"It works on my machine" is not on this list, and never satisfies it.

---

## 11. Changing This Document

This document changes when a rule proves wrong in practice, not when it proves
inconvenient. A rule bypassed once is a rule that needs amending or deleting — a rule that
exists but is routinely ignored is worse than no rule, because it teaches that the
document is optional.

Amendments are made by editing this file. Git history is the version record; this document
carries no version number.
