---
name: diagnose
description: Find the cause of a bug before fixing it. Use when a test, a check or reported behaviour is wrong and the error does not name the cause; when a fix was tried and the failure is still there; when a check passed at BASE and fails now and the change responsible is unknown; or when a bug report cannot yet be turned into a failing test. Used by the builder inside its task worktree and by a single session fixing a bug. Not when the error message already names the cause, and not for the reviewer, which reports defects and does not fix them.
---

# diagnose

The result of a diagnosis is a test that fails because of the cause, and then the fix. No
product code changes until that test exists.

A builder works only in its worktree and file scope. When the cause lies outside the scope,
that is `BLOCKED`, not a reason to widen it.

**Safety while diagnosing.** Reproductions and experiments run against test data or a copy,
never the data the application uses for real. A step with an outside effect — a message
sent, a payment, a paid call, a change to a real service — is explained and approved first,
as the project rules require for any such action. Temporary output never prints a secret or
a person's data. Use the tools the project already has; installing one is a dependency
change.

Before the first temporary change, note `git status --porcelain`. Undo only what you added;
changes that were there before are not yours to touch.

## 1. Reproduce

1. Write down what was expected, what happened, the input, and the command that shows it.
   Keep that command and the original error output: they go with any stop (§5).
2. Find the smallest reproduction that still fails the same way and goes through the same
   path as the report — through the task's seam under test where there is one, because that
   is where the test will go.
3. Confirm it fails every time. If it fails only sometimes, find the condition that makes it
   fail every time — order, timing, a random seed, the date or time zone, concurrency, data
   already stored. No such condition means it is not reproduced yet: step 5.
4. Check the environment as part of the reproduction: tool and library versions, the
   configuration, which environment variables are set (names only, never values), the data
   used.
5. If it cannot be reproduced, change nothing and stop (§5), saying what was tried and what
   would let it reproduce — an input, a log, an environment.

## 2. Narrow

Find where a correct value first becomes wrong, using whichever of these fits:

- **Input.** Remove parts of the input while it still fails, until removing anything more
  makes it pass.
- **History.** When a known commit works, the reproducing command runs on the commits since,
  and it has no outside effect, `git bisect` finds the first bad commit:
  - only with a clean tree (`git status --porcelain` empty);
  - `git bisect start <bad> <good>`, then `git bisect run <reproducing command>` — exit 0
    means good, 125 skips a commit that cannot be tested, anything else means bad;
  - always end with `git bisect reset`, and confirm that the branch
    (`git symbolic-ref --short HEAD`) and HEAD are what they were before you edit anything.
- **Path.** Follow the value from the input to the wrong output. Put temporary output — a
  print, a log line, an assertion — halfway along the path, run, and keep the half where it
  goes wrong. Repeat.

Stop narrowing when what you have found explains every observation. That is usually one
place — a function, a condition, a line — and sometimes two places that are each correct
alone but wrong together. That place, or pair, is the cause under investigation.

## 3. One guess, one experiment

1. State the guess in one line: what is wrong there, and why that gives the observed output.
2. Name a result the guess predicts that you have not seen yet, and an experiment that comes
   out differently if the guess is wrong. Run it. A temporary code change for the experiment
   is undone afterwards.
3. Wrong → record an attempt (§5) and make the next guess from what the experiment showed.
   Right → §4.

## 4. Test, then fix

1. Write the test at the seam. It reproduces the bug, fails, and the failure comes from its
   assertion, for the reason the guess names. If no seam can capture the failure in a test,
   stop (§5) and say why: the human decides how it is verified.
2. Change the code at the cause, not where the symptom shows. Never change the expected
   value, skip the test, swallow the error, or special-case the input that reproduces it.
3. The test passes. Undo the fix and it fails again; restore the fix. A test that passes
   either way does not exercise the cause.
4. Run every check command. If the test still fails or a check breaks that passed before,
   the fix failed: record an attempt, undo the change, and go back to §3 with what it
   showed.
5. Remove all temporary output. `git status --porcelain` and `git diff` show nothing of it
   beyond the test and the fix. Commit the test and the fix together.

## 5. Counting and stopping

- An attempt is a wrong guess (§3) or a failed fix (§4) at the cause under investigation.
  Record each as `cause / change / observation`.
- The count belongs to the place found in §2. A new guess about the same place adds to it.
  It starts again only when narrowing or an experiment shows the fault is at a different
  place. A builder counts the earlier attempts its prompt gives at the same place, unless a
  human direction recorded after them starts a new count.
- At three attempts, do not guess a fourth time. Go back to §2 and narrow further with
  whatever has not been used yet — a bisect, more output points along the path, a smaller
  input. If that shows a different place, it is a new cause: continue at §3. If nothing
  narrows further, stop.
- Stopping goes to the human, never to another agent or a session with wider permissions.
  Every stop carries the reproducing command, the original error and the attempts.
  - A builder returns `STRUCK_OUT` with every attempt under `ATTEMPTS`, and under `NEEDS`
    the questions for the human. When it cannot reproduce (§1) or cannot capture the failure
    in a test (§4), it returns `BLOCKED` with what is needed under `NEEDS`.
  - A single session puts one decision to the human, in their language, with the options
    that fit. The usual ones:

    ```
    <the bug> — three guesses at <place> were wrong, and it cannot be narrowed further
    A. Answer these questions — <questions> — and the search continues from the answers
    B. Start again in a new session, with these attempts given to it
    C. Defer it — a line in known-issues.md, and other work goes first
    Suggested: <option>, because <reason>
    ```

- Ask only questions whose answer would change the next step, and that a non-developer can
  answer from what they saw: when it was first seen, whether it happens for every user,
  record or machine, what changed around that time, what the screen shows. Never ask the
  human to find the cause in the code. If no such question exists, offer no A.
