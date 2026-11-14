# Pre-mortem: my first CD pipeline to Render

15 Nov 2026 · [KhanalChess](https://github.com/exosphere8/KhanalChess) ·
written before any deploy code exists (main at 87eeb99)

## The setup

CI is finished: every pull request builds, tests and lints, and a ruleset
blocks the merge if any check fails. The next stage is CD, continuous
deployment. After a merge to `main`, a pipeline should build the Docker image,
run it on a host, and smoke test `/health` and `/health/db` on the live
address.

The decision record (`docs/cd-decision.md`) proposes Render's free tier first,
to prove the pipeline with no servers to administer, and AWS with Terraform
later. This post lists what I expect to go wrong, so that after the deploy I
can check each prediction against what actually happened. Facts about Render
were checked on 2026-11-15 on its documentation and must be re-checked.

## Predictions

| # | I expect | Why | How I will check | Result |
|---|---|---|---|---|
| 1 | The first smoke test after a deploy fails | The free service spins down after 15 minutes without traffic and needs about a minute to start | Run the check once without a retry, then with a retry and a timeout | |
| 2 | `/health/db` returns 503 at first | The database variables Render provides probably do not match the `POSTGRES_*` names my code reads | Call `/health` and `/health/db` and read the status line with `curl.exe -i` | |
| 3 | The free database stops working 30 days after creation | Free Render Postgres expires after 30 days, with a 14-day grace period before deletion | Write down the creation date, then check both endpoints near the date | |
| 4 | The sign-up asks for a card | Third-party articles say no, Render's pricing page mentions a $1 check; my card is a prepaid virtual card | Read what the page says before I enter anything | |
| 5 | A secret ends up in git or in a log | Deploy jobs need tokens, and I have a `.env` file with real values | `git status --short` before staging; secrets only in GitHub environment secrets | |
| 6 | Action versions I remember are out of date | Versions change quietly, and I checked the CI ones on 2026-11-14 only | Look up the current versions before writing the workflow | |
| 7 | The first image push is refused (low confidence) | If I use a registry, the workflow token may lack the permission to write packages | Read the error and the job permissions | |

## What I am not predicting

The ephemeral filesystem should not matter, because the API writes no files.
The free database expiring should not lose data, because the app stores none
yet. If either turns out to matter, that is a surprise and belongs in the
update below.

## Result

*To be filled in after the deploy:* [FILL IN: for each row, what actually
happened, which predictions were wrong, and which wrong turns were not on the
list at all.]
