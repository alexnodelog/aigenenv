---
name: discover
description: Run the Discover phase — interview the human one question at a time and write PRD.md until it has no open question, then get it approved. Use when the human starts a new Standard feature or project and no approved PRD.md covers it; when the human asks to write, revise or reopen PRD.md; when state.md's Plan shows a planner question starting "PRD:"; or when a change would add scope that PRD.md does not contain. Not for Small tasks, and never inside an orchestrator session.
---

# discover

Discover is a conversation, so it runs in its own session with the human and ends when
`PRD.md` is approved. Its output is `PRD.md` and the approval line in `state.md`. A builder
later writes tests from what this file says, so every requirement has to be checkable
without asking anyone.

## 1. What this session touches

- Reads: the human's request, the existing `PRD.md`, `state.md`, and code only to answer a
  factual question.
- Writes: `PRD.md`, and `state.md` through skill `handoff`.
- Never writes code, tests, `PLAN.md` or a prototype, and never names a framework, library
  or design as a requirement. A constraint the human states — "must run with no network",
  "Windows PCs only" — is recorded as a constraint.

## 2. Start

1. Run skill `handoff` §7 if `state.md` exists.
2. Find the reason this session exists:
   - **New work:** the human's request.
   - **Reopened by the planner:** the `PRD:` items under `state.md` `## Plan` → `Needs`.
     Only those questions are reopened.
   - **Revision asked by the human:** the change they described.
3. Read `PRD.md` if it exists. If it is approved, every change here needs a new approval.
4. Look up every fact the code or the files can answer — what exists, what it is called,
   what it does now — before forming any question.

## 3. PRD.md

```markdown
# PRD: <name>

## Problem
<what is wrong or missing today, for whom, in the human's terms>

## Users
- <who uses it, how often, on what device or setting>

## Requirements
### R1: <one behaviour, one line>
- Acceptance: <input or situation> → <expected result>; at least one worked example
- Limits and errors: <input limits, error cases and what the user sees | none stated>

## Constraints
- <stated by the human: where it runs, network, data it may hold, deadlines | none>

## Out of scope
- <item> — <the human's reason | unanswered, YYYY-MM-DD>

## Open questions
- <question>
```

- Requirement ids (`R1`, `R2`, …) are never reused or renumbered; the planner and every
  commit trace to them. A dropped requirement moves to `## Out of scope`.
- Acceptance examples use concrete values. They are the source of the builder's expected
  results, so "works correctly" is not an example.
- `## Open questions` exists only while drafting. The file is not offered for approval
  until that section is gone.

## 4. Interview

1. Write the draft from the request, putting every unknown under `## Open questions`. An
   unknown is never filled with a plausible guess.
2. Ask one question at a time, the ones that change scope first. Each question says why it
   matters and, where there are options, what each would change. Never ask what §2 step 4
   could look up.
3. Write each answer into `PRD.md` as soon as it is given, and remove the question. The
   session may end at any point.
4. When the human cannot or will not answer, say that the item will be written as out of
   scope, and move it to `## Out of scope` marked `unanswered` with the date. Never leave it
   in the file as deferred.
5. Collecting, storing or sending sensitive personal data — national ID numbers, health,
   biometric or financial account data — is the human's decision, asked as its own
   question. Record the answer under `## Constraints`; the task that builds it writes the
   ADR.
6. Before the session might run out of room, run skill `handoff` with
   `## Next`: `Continue discover: <the next open question>`.

## 5. Finish

1. Check `PRD.md`:
   - `## Open questions` is gone.
   - Every requirement has an acceptance example with concrete values.
   - `## Out of scope` is present, even if it holds only `none`.
   - No requirement names a framework, library or design.
2. Commit it alone: `git add PRD.md`, then
   `git commit -m "docs(prd): <what this version settles>"`.
3. Ask the human to approve `PRD.md` as committed, naming the short SHA. Approval is the
   human saying so; silence or "looks fine so far" is not.
4. When they approve, record it with skill `handoff` §5, on the branch tasks will merge
   into, and set `## Next` to `Start orchestrate in a new session`. If this session was
   reopened by the planner, set the plan `Status` to `blocked` with `Needs: none — PRD
   re-approved, respawn the planner`.
5. Tell the human to end this session and start the next one with `orchestrate`.

A change to `PRD.md` after approval goes through §4 and §5 again; the old approval line is
kept as superseded.
