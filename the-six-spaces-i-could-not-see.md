# The six spaces I could not see

14 Nov 2026 · [KhanalChess](https://github.com/exosphere8/KhanalChess) ·
found while editing `ci.yml` for the requirements split (PR #18)

## The setup

For the requirements split, the test job in `.github/workflows/ci.yml` had to
install `requirements-dev.txt` instead of `requirements.txt`, and its cache
key had to list both files. A small edit to one YAML file, in VS Code.

## The wrong turns

**1. "I changed the file, so the file is changed."**
I pasted the old contents of `ci.yml` into the edit, and the first edit was
not saved. Nothing on the screen looked wrong. [FILL IN: how I noticed.] The
editor shows an unsaved file with a dot on its tab. I had not looked for the
dot.

**2. "A final Enter is harmless."**
I pressed Enter at the end of the last line. VS Code auto-indented the new
line by 6 spaces to match the block above, and the file then ended with six
invisible spaces and no final newline. Nothing in the editor showed it.

## What was actually happening

The two problems had the same cause: I was trusting what the editor showed
instead of what was on disk.

- `git diff` reports a missing final newline with the line `\ No newline at
  end of file`. That line is a change unrelated to the one I meant to make.
  It would have been a second idea inside a one-idea pull request.
- `Format-Hex <file> | Select-Object -Last 2` shows the last real bytes of
  the file. Spaces (0x20) where a line feed (0x0A) belongs settle the
  question without guessing.

I fixed the ending before committing, so the PR contained only the intended
change.

## The takeaway

Before staging, read the diff for `No newline at end of file`, check the
tab for an unsaved dot, and look at the last bytes with `Format-Hex` when
something might be invisible. For small edits I later switched to a
safe-replace pattern in PowerShell: read the file, count the matches of the
old text, and write only if the count is exactly what I expect; otherwise
print "nothing changed". One idea per pull request starts with seeing the
whole diff, including the parts the editor hides.
