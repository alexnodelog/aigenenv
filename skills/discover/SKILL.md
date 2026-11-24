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
<what is wrong or missing today, for whom, and what they do instead now>

## Users
- <who uses it, how often, on what device or setting>

## Requirements
### R1: <one behaviour, as the user sees it>
- Acceptance: <input or situation> → <expected result>; at least one worked example
- Limits and errors: <input limits, error cases and what the user sees | none stated>

## Constraints
- <stated by the human: where it runs, network, data it keeps, deadlines | none>

## Out of scope
- <item> — not wanted | later | unanswered, YYYY-MM-DD

## Open questions
- <question>
```

- `PRD.md` is written in the language the human uses, so they approve what they can read.
  Headings and requirement ids stay as in the template.
- Requirement ids (`R1`, `R2`, …) are never reused or renumbered; the planner and every
  commit trace to them. A dropped requirement moves to `## Out of scope`.
- Acceptance examples use concrete values. They are the source of the builder's expected
  results, so "works correctly" is not an example.
- A quality the human wants — fast, easy, secure, reliable — is written with a number or
  an example ("results within 2 seconds for 10,000 rows"), or not at all.
- What the human wants later, not now, goes to `## Out of scope` as `later`. It returns
  only through a new discover session.
- `## Open questions` exists only while drafting. The file is not offered for approval
  until that section is gone.

## 4. Interview

1. Write the draft from the request, putting every unknown under `## Open questions`. An
   unknown is never filled with a plausible guess. Check the request against each area
   below and add a question for every one it leaves open:

   | Area | What has to be known |
   |---|---|
   | Problem | What goes wrong today, and what the user does instead |
   | Users | Who, how many, how often, on what device; whether they sign in |
   | Core actions | What the user does, step by step, and what they get back |
   | Data | What is entered, shown and kept; for how long; whether it must survive a reinstall; whether any of it is sensitive personal data |
   | Where it runs | PC, browser or phone; with or without a network; who installs it |
   | Outside services | Anything it sends to or receives from another system; any cost or message it can cause |
   | Errors and edges | Empty input, wrong input, too much input, no network, a failed step halfway |
   | Look | Existing screens, mockups or examples to follow, or none |
   | Model answers | Only when the product answers with a model: how a right answer is judged, with example questions and expected answers |
   | Not wanted | What it must not do, and what can wait |

2. Ask one question at a time, the ones that change scope first. Never ask what §2 step 4
   could look up. Give the human choices to pick from:

   ```
   <question> — <why it matters, one line>
   A. <option> — <what it changes>
   B. <option> — <what it changes>
   C. Something else
   Suggested: B, because <reason>.
   ```

   A free answer is accepted as well; the choices only make answering easier.
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
