# Green is not the same as seen

15 Nov 2026 · [KhanalChess](https://github.com/exosphere8/KhanalChess) ·
PRs #18 to #23, merged between 2026-11-14 23:57 and 2026-11-15 01:30 (+0545)

## The setup

KhanalChess has three required CI checks: `test`, `docker` and `lint`. The
ruleset greys out "Squash and merge" until all three pass. My own rule for
each pull request was: wait for the checks, show them to my mentor, then
merge. For six pull requests in a row (#18 to #23) I merged first and showed
nothing.

## The wrong turns

**1. "The merge button is enabled, so the checks passed."**
This is actually true. The ruleset requires all three checks, so an enabled
button means they were green. [FILL IN: why I skipped looking anyway.] I was
not merging broken code. That is why the habit felt harmless.

**2. "Then there is nothing to look at."**
Wrong, and I only found out later, when I finally opened the Actions run
list:

- Main runs for #19 to #22 were all green, 22 to 24 s each. Good news I had
  not checked.
- On the PR run for #22, `lint` took 16 s, where it had taken 7 s before. The
  lint job now installs all of `requirements-dev.txt` to read the ruff version
  from one place, and it has no pip cache. A real cost of my own change, and
  invisible behind a green tick.
- PR #21 pinned the runner to `ubuntu-26.04`. A green run suggests the label
  resolved, but only the "Set up job" log names the image that actually ran.
  I had never read it.
- The PR run for #18 and the checks for #23 had still not been looked at when
  I wrote this.

## What was actually happening

A required check is a guarantee that something passed. It is not a record of
what passed, how long it took, or what it ran on. The gate protected `main`.
It did not tell me anything about my own pipeline. Because I skipped looking,
the evidence stayed missing, and the gaps list for the project carried the
same items from one report to the next.

## The takeaway

The green tick answers "may I merge?". Looking answers "what did I just
build?". For every pull request, I open the checks, read the times and the
jobs, and show them before merging. When I want a claim about the pipeline in
a report, I attach the run that proves it.

*Update, once seen:* [FILL IN: result of checking the main run for the #23
merge (87eeb99) and the "Set up job" lines for ubuntu-26.04.]
