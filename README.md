# postmortems

Debugging post-mortems. Mostly about being wrong for a while before being right.

Tutorials show the straight path from problem to fix. These write-ups keep the detours: each
hypothesis I held, why it was plausible, the evidence that killed it, what was actually going
on, and the habit I am changing because of it. Most of the time in debugging goes into those
detours, so that is where the lessons are.

| Date | Post | In one line |
|---|---|---|
| 19 Sep 2026 | [The numpy version was never the problem](numpy-was-never-the-problem.md) | Three crashes, three wrong hypotheses, and a clue I ignored for an hour. |
| 20 Sep 2026 | [The penalty I was terrified of cost me 0.45%](the-penalty-that-cost-0.45-percent.md) | Reasoning from a formula, with a number I'd invented. |

## How each post is written

- **The setup:** enough context to follow along, nothing more.
- **The wrong turns, in order,** including what made each one look right at the time.
- **What was actually happening,** and the evidence that proved it.
- **The takeaway:** one rule I am applying from now on.

They are blameless by design: the point is to understand the mistake, not to perform modesty
or hindsight.

---

Written content is licensed under [CC BY 4.0](LICENSE); code snippets are also available under MIT.
