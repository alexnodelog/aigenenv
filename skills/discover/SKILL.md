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
3. Read `PRD.md` if it exists. `git status --porcelain PRD.md` must be empty: changes this
   session did not make are the human's — stop and ask. If `PRD.md` is approved, from the
   first edit here the approval no longer covers it, and it needs a new approval (§5).
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
- Limits and errors: <input limits, error cases and what the user sees | none>

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
- A quality the human wants — fast, easy, secure, reliable — becomes a requirement only in
  a checkable form the human agrees to: a number ("results within 2 seconds for 10,000
  rows"), an example ("a first-time user finds the export button without help"), or a rule
  that must always hold with how to check it ("one user never sees another's records:
  signed in as A, asking for B's record is refused"). A wish in none of these forms stays in
  `## Problem` as context, not as a requirement.
- `## Out of scope` holds three different things: `not wanted` is a decision; `later` is
  wanted but not now; `unanswered` is undecided (§4 step 4). A `later` or `unanswered` item
  returns only through a new discover session.
- `## Open questions` exists only while drafting. The file is not offered for approval
  until that section is gone.

## 4. Interview

1. Write the draft from the request, putting every unknown under `## Open questions`. An
   unknown is never filled with a plausible guess. Go through the areas below. Ask about an
   area only when its answer could change a requirement, an acceptance example, a constraint
   or a risk, and neither the request, the existing `PRD.md` nor the code already settles
   it. A small change to an existing product usually touches two or three areas, not all:

   | Area | What has to be known |
   |---|---|
   | Problem | What goes wrong today, and what the user does instead |
   | Users | Who, how many, how often, on what device; whether they sign in; whether they see each other's data, and who may change or delete what |
   | Core actions | What the user does, step by step, and what they get back |
   | Data | What is entered, shown and kept; for how long; whether it must survive a reinstall; what deleting it removes; whether it can be exported or restored; whether any of it is sensitive personal data |
   | Where it runs | PC, browser or phone; with or without a network; who installs it |
   | Outside services | Anything it sends to or receives from another system; any cost or message it can cause; what the user sees when the call fails or its result is unknown |
   | Errors and edges | Empty input, wrong input, too much input, no network; the same action twice or at once; dates and times — whose time zone; a step that fails halfway — what is kept and whether it can be run again |
   | Look | Existing screens, mockups or examples to follow, or none |
   | Model answers | Only when the product answers with a model: how a right answer is judged, with example questions and expected answers |
   | Not wanted | What it must not do, and what can wait |

2. Ask one question at a time, from the open questions whose answer does not depend on
   another open one. Among those, ask first about data, sensitive personal data, and
   anything with an effect outside the product; then the rest. A question that depends on
   an unanswered one waits, and each answer can open new questions — add them to the list.
   Never ask what §2 step 4 could look up. Where the question has real alternatives, give
   the human choices to pick from; otherwise ask it plainly:

   ```
   <question> — <why it matters, one line>
   A. <option> — <what it changes>
   B. <option> — <what it changes>
   C. Something else
   Suggested: B, because <reason>.
   ```

   A free answer is accepted as well; the choices only make answering easier. A suggestion
   is given only when it rests on something stated — the code, the project rules, an earlier
   answer — and says what. A question only the human can know the answer to — who the users
   are, what they do today, what they want — gets choices but no suggestion.
3. Write each answer into `PRD.md` as soon as it is given, and remove the question. The
   session may end at any point.
4. When the human cannot or will not answer, the item goes to `## Out of scope` marked
   `unanswered` with the date — and so does every requirement that cannot be written
   without that answer. Say which requirements go with it before moving them. If nothing
   is left that can be built, say so: discover cannot finish until the human answers.
   Never leave the item in the file as deferred.
5. Collecting, storing or sending sensitive personal data — national ID numbers, health,
   biometric or financial account data — is the human's decision, asked as its own
   question. If they say yes, ask in turn: why it is needed, the least that will do, where it
   is kept, who may see it, how long it is kept and how it is deleted, and whether it ever
   leaves the product. Each answer becomes a requirement or a constraint; an unanswered one
   follows step 4. The task that builds it writes the ADR.
6. Whenever the session stops before approval — the human pauses, the session is running
   out of room, or anything else — run skill `handoff` with
   `## Next`: `Continue discover: <the next open question>`.

## 5. Finish

1. Check `PRD.md`:
   - `## Open questions` is gone.
   - Every requirement has an acceptance example with concrete values, and none of them
     depends on an item marked `unanswered`.
   - Every requirement's `Limits and errors` says what happens, or `none` after the
     relevant areas were considered.
   - `## Out of scope` is present, even if it holds only `none`.
   - No requirement names a framework, library or design.
2. Commit it alone. `git diff --cached --name-only` must print nothing before you start; if
   it lists files, stop and ask, since they are not this session's. Then `git add PRD.md`,
   confirm the same command now prints only `PRD.md`, and
   `git commit -m "docs(prd): <what this version settles>"`.
3. Ask the human to approve `PRD.md` as committed, naming the short SHA: the approval is
   for the whole file at that commit. Approval is the human saying so without conditions;
   silence, "looks fine so far" or "approved, but …" is not. An unclear reply — "nice",
   "should be fine" — is answered by asking once whether that is approval.
4. When they approve, record it with skill `handoff` §5, on the branch tasks will merge
   into, and set `## Next` to `Start orchestrate in a new session`. If this session was
   reopened by the planner, set the plan `Status` to `blocked` with `Needs: none — PRD
   re-approved, respawn the planner`.
5. Tell the human the next step is `orchestrate` in a new session, now or whenever they
   are ready.

A change to `PRD.md` after approval goes through §4 and §5 again. The new commit is made
first, then the new approval is recorded in `state.md` (skill `handoff` §5), which keeps the
earlier approval line there marked superseded.
