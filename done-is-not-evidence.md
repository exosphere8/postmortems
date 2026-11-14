# "Done" is not evidence

14 Nov 2026 · [KhanalChess](https://github.com/exosphere8/KhanalChess) ·
written from my session notes of 13 to 15 Nov 2026

## The setup

I was building the CI pipeline and the database check for KhanalChess, working
step by step with a mentor who read my terminal output and told me what it
meant. That only works if the output actually reaches the reader. Several
times it did not, and each time the gap cost me a re-run.

## The wrong turns

**1. "It passes" instead of the output.**
I answered "passes" about the tests and the containers without pasting the
output of `pytest` or `docker ps -a`. [FILL IN: what I believed I had
checked.] A word is a claim, and a claim nobody can read is not a check.

**2. A wrong name, repeated.**
I typed `ci` instead of `ci.yml` and a branch name that did not exist
(`test/add-lint`). The same failing `git diff` command then ran three times
before I read the error.

**3. A whole history instead of the newest output.**
In several messages I pasted the entire terminal history. The newest lines,
the only ones that matter, were buried under commands I had already run.

**4. Cleanup nobody saw.**
After closing the throwaway PR #16 I deleted the branch and pruned the
remote, but did not show the output. Those commands had to be run again so
the result could be read.

**5. An edit that was never saved.**
I pasted the old `ci.yml` and made an edit, and the first edit was not saved.
I would have gone on believing the file had changed.

## What was actually happening

Nothing was dramatically broken. The pattern was smaller and more constant:
I reported status words ("done", "as expected", "all passed") where the
evidence should have been. Each one was probably true. But "probably true"
cost more than the output would have, because every unverified step was a
place where a wrong filename, an unsaved file or a failed command could sit
unnoticed until the next step stood on it.

## The takeaway

A status word is not a status. I paste the newest output with every claim:
the real command, the real result, nothing older. If I catch myself typing
"done" or "as expected", the next thing I do is paste the output that proves
it.
