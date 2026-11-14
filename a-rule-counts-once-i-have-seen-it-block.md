# A rule counts once I have seen it block

14 Nov 2026 · [KhanalChess](https://github.com/exosphere8/KhanalChess) ·
throwaway PRs #13 (failing test) and #16 (unused import), both closed
without merging

## The setup

The repository has a ruleset called `protect-main`. It requires a pull
request, allows squash merges only, blocks force pushes, and requires the CI
checks `test`, `docker` and `lint` to pass. It also requires the branch to be
up to date before merging. On the settings page all of this showed as active.

## The wrong turns

**1. "The ruleset lists the checks, so a failing check blocks the merge."**
It is a reasonable belief: the settings page says the rule is on. But I had
only ever read the configuration. I had never watched it stop anything.
[FILL IN: what made me want to test it anyway.]

**2. "Adding lint to the list is enough."**
When I wanted `lint` to be required, I found two details. A required check
can only be chosen in the ruleset after it has run at least once, so it was
not selectable until the lint job had a run. And a ruleset change must be
saved, then the page reloaded, to confirm it really stuck. A change that
looks applied on screen is not the same as a change that is saved.

## What was actually happening

The rule worked as configured, and the only way to know was to try to break
it on purpose.

- **PR #13** contained a deliberately failing test. The `test` check turned
  red and "Squash and merge" was greyed out.
- **PR #16** contained an unused import. Only `lint` turned red (7 s). `test`
  and `docker` stayed green, and "Squash and merge" was still greyed out.

The second PR taught more than the first. It showed that `lint` blocks on
its own, not only when something else fails. There was no owner bypass
either: as the repository owner, I could not push the button.

Both PRs were closed, never merged, and their branches were deleted. For a PR
closed without merging there is nothing to compare, so the usual `git diff
main <branch>` check does not apply (`unknown revision` simply means the
branch is already gone). Instead I checked that `main` was still at the same
commit, `3db4b12`.

## The takeaway

A control counts once I have watched it block. After adding or changing a
required check, I open a throwaway PR that breaks exactly that check, look at
the merge button, then close the PR without merging.
