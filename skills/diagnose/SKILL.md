---
name: diagnose
description: Find the cause of a bug before fixing it. Use when a test, a check or reported behaviour is wrong and the error does not name the cause; when a fix was tried and the failure is still there; when a check passed at BASE and fails now and the change responsible is unknown; or when a bug report cannot yet be turned into a failing test. Used by the builder inside its task worktree and by a single session fixing a bug. Not when the error message already names the cause, and not for the reviewer, which reports defects and does not fix them.
---

# diagnose

The result of a diagnosis is a test that fails because of the cause, and then the fix. No
product code changes until that test exists. Every step below either narrows where the fault
is or tests one guess about it; a guess is cheap to make and expensive to keep making, so
the steps that need no guess come first.

A builder works only in its worktree and file scope. When the cause lies outside the scope,
that is `BLOCKED`, not a reason to widen it.

## 1. Reproduce

1. Write down what was expected, what happened, the input, and the command that shows it.
2. Make it fail on demand with the shortest command you can find — through the task's seam
   under test where there is one, because that is where the test will go.
3. Run it again. If it fails only sometimes, find the condition that makes it fail every
   time — order, timing, a random seed, the date or time zone, concurrency, data already
   stored — before going on.
4. Check the environment as part of the reproduction: tool and library versions, the
   configuration, which environment variables are set (names only, never values), the data
   used.
5. If it cannot be reproduced, change nothing and stop (§5), saying what was tried and what
   would let it reproduce — an input, a log, an environment.

## 2. Narrow

Find the first place where a correct value becomes wrong. Use whichever of these applies:

- **Input.** Remove parts of the input while it still fails, until removing anything more
  makes it pass.
- **History.** When a known commit works, find the first bad one with `git bisect`:
  - only with a clean tree (`git status --porcelain` empty);
  - `git bisect start <bad> <good>`, then `git bisect run <reproducing command>` — exit 0
    means good, 125 skips a commit that cannot be tested, anything else means bad;
  - always end with `git bisect reset`, and confirm that the branch
    (`git symbolic-ref --short HEAD`) and HEAD are what they were before you edit anything.
- **Path.** Follow the value from the input to the wrong output. Put temporary output — a
  print, a log line, an assertion — halfway along the path, run, and keep the half where it
  goes wrong. Repeat.

Temporary output is never committed. Remove it before staging, and check `git diff --cached`
for any that remains.

Stop narrowing when one place — a function, a condition, a line — explains the failure. That
place is the cause under investigation.

## 3. One guess, one experiment

1. State the guess in one line: what is wrong at that place, and why that gives the observed
   output.
2. Name a result the guess predicts that you have not seen yet, and an experiment that comes
   out differently if the guess is wrong. Run it. A temporary code change for the experiment
   is undone afterwards.
3. Wrong → record an attempt (§5) and make the next guess from what the experiment showed.
   Right → §4.

## 4. Test, then fix

1. Write the test at the seam. It reproduces the bug, fails, and the failure comes from its
   assertion, for the reason the guess names.
2. Make the smallest change at the cause that makes it pass. Never change the expected value,
   skip the test, swallow the error, or special-case the input that reproduces it.
3. Run every check. If the test still fails or another check breaks that passed before, the
   fix failed: record an attempt, undo the change, and go back to §3 with what it showed.
4. Remove all temporary output. Commit the test and the fix together.

## 5. Counting and stopping

- An attempt is a wrong guess (§3) or a failed fix (§4) at the cause under investigation.
  Record each as `cause / change / observation`. Running the same thing again is not an
  attempt.
- The count belongs to the place found in §2. A new guess about the same place adds to it.
  It starts again only when narrowing or an experiment shows the fault is at a different
  place. A builder counts the earlier attempts its prompt gives at the same place, unless a
  human direction recorded after them starts a new count.
- At three attempts, do not guess a fourth time. Go back to §2 and narrow further with
  whatever has not been used yet — a bisect, more output points along the path, a smaller
  input. If that shows a different place, it is a new cause: continue at §3. If nothing
  narrows further, stop.
- Stopping goes to the human, never to another agent or a session with wider permissions:
  - A builder returns `STRUCK_OUT` with every attempt under `ATTEMPTS`, and under `NEEDS`
    the questions of fact the human can answer. When it cannot reproduce (§1), it returns
    `BLOCKED` with what would let it reproduce under `NEEDS`.
  - A single session puts one decision to the human, in their language:

    ```
    <the bug> — three guesses at <place> were wrong, and it cannot be narrowed further
    A. Answer these questions — <two or three questions of fact> — and the search continues from the answers
    B. Start again in a new session, with these attempts given to it
    C. Defer it — a line in known-issues.md, and other work goes first
    Suggested: <A, B or C>, because <reason>
    ```

- Questions of fact are ones a non-developer can answer: when it was first seen, whether it
  happens for every user, record or machine, what changed around that time, what the screen
  shows. Never ask the human to find the cause in the code.
