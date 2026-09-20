# The penalty I was terrified of cost me 0.45%

*On the difference between knowing what a metric penalizes and knowing how much.*

*20 September 2026*

---

## The setup

I'm competing in a cell-tracking competition: detect cells in 3D microscopy
volumes across 100 timepoints, link them frame to frame, output a graph. The
evaluation page describes the score in one sentence:

> The edge Jaccard is TP / (TP + FP + FN), adjusted by a penalty on
> over-predicting the total number of nodes.

My model had just produced its first real output: **26,905 detected cells** in a
single video. That seemed like a lot. And the metric penalizes over-prediction.
So before running anything else, I went looking for how bad the damage was.

## Reasoning from the formula

I found the penalty in the source:

```python
total_node_ratio = (n_pred - n_total) / n_total
adj_edge_jaccard = max(0.0, edge_jaccard * (1 - 0.1 * total_node_ratio))
```

Straightforward. `n_total` is the estimated true number of cells. Predict more
than that and your Jaccard gets scaled down by a tenth of the excess ratio.

So I did what felt like diligence: I picked a plausible value for `n_total` and
worked out the consequence. The ground-truth annotations in that video contained
52 nodes. Annotations are sparse — the organizers say so explicitly — so the
true count is obviously much higher. Call it 5,000.

```
total_node_ratio = (26905 - 5000) / 5000 = 4.38
multiplier       = 1 - 0.1 × 4.38 = 0.56
```

A 44% haircut. And it got worse: at a ratio of 10, the multiplier hits zero.
Predict 55,000 cells against a true 5,000 and your score is **zero**, no matter
how perfectly you tracked them.

This felt like the most important finding of the day. Detection threshold
suddenly looked like the dominant lever — more important than model quality,
more important than the linking. I was ready to reorganize the whole plan around
pruning detections.

## The number I hadn't looked up

`n_total` isn't a mystery. It ships with the data, in the GEFF metadata, as
`estimated_number_of_nodes`. Ten seconds of reading rather than guessing:

```
44b6_0113de3b  25755
44b6_0b24845f  32795
44b6_0c582fdc  27958
44b6_0db75fae  15335
44b6_12dfb391  58672
```

The video where I predicted 26,905 has a true estimate of **25,755**.

```
total_node_ratio = (26905 - 25755) / 25755 = 0.045
multiplier       = 1 - 0.1 × 0.045 = 0.9955
```

The catastrophic penalty cost me **0.45%**.

My guess of 5,000 was off by a factor of five, and because the penalty is linear
in the ratio, the error propagated straight into a conclusion that was wrong by
roughly a hundredfold. I had spent real thought on a lever that turns out to be
almost flat near my operating point.

## The second thing I was wrong about

While I was at it, I'd been quietly worried about something else. Ground truth
is sparse — 52 annotated nodes in a video containing ~25,755 real cells. My
model was emitting **24,607 edges**. If the unmatched ones counted as false
positives, then:

```
jaccard ≈ 50 / 24607 ≈ 0.002
```

Everyone's score would be approximately zero, which is not how competitions are
built. So either I was misreading it, or something in the implementation handled
it. I went and read.

```python
pred_valid = out_valid | in_valid
...
denom = gt_num_edges + n_valid_pred_edges - intersection
```

A predicted edge only enters the denominator if one of its endpoints matched an
*annotated* ground-truth node. Everything predicted in unlabelled regions is
excluded — neither rewarded nor punished. Sparse annotation is handled by
scoring you only where annotation exists, with the node-ratio adjustment acting
as the separate, global check on over-prediction.

So the 24,000 edges I was anxious about are, to a first approximation, free.

## What both mistakes have in common

The evaluation page was accurate. It said there's a penalty on over-predicting
nodes, and there is. It said the metric accounts for sparse labels, and it does.
Nothing I read was wrong.

What prose can't tell you is **magnitude**, and magnitude is the entire content
of a prioritization decision. "There is a penalty" and "the penalty currently
costs you 0.45%" point at completely different days of work. The first says
restructure everything around detection count. The second says stop thinking
about detection count and go look at the linking.

The gap between them was one measurement I could have taken in ten seconds,
against a number sitting in a metadata field I already had on disk.

I notice I substituted a guess for that lookup without registering it as a
guess. "Call it 5,000" felt like a modelling assumption — reasonable, roughly
the right shape. It was actually the single most load-bearing quantity in the
calculation, and I invented it. The arithmetic afterwards was flawless and
completely worthless, which is the characteristic failure mode: rigour applied
downstream of a number nobody checked.

## The rule I'm taking from it

Before optimizing against a metric, implement it — or at least evaluate it — on
your own real output. Not the formula. The formula with your actual numbers in
it.

Two questions, both cheap:

**What are my current values for every term?** Not plausible values. Actual
ones. Anything you can't measure yet is a thing to go measure, not a thing to
estimate.

**How much does the score move if this term changes?** A penalty that costs
0.45% at your operating point is not a lever, whatever the documentation says
about it. A term you can't move is not a lever either. Only the intersection
deserves your week.

I got lucky here in a specific way worth naming: the wrong conclusion would have
sent me pruning detections, which would have *lowered* my score — fewer
detections means fewer true-positive edges, and the penalty I'd have been
avoiding wasn't costing me anything. The measurement didn't just refine my
priorities, it reversed their sign.

The next thing I build is a local evaluation harness: run the real metric
against held-out labelled videos, so every future question — threshold,
activation, whether to allow divisions, greedy versus a global solver — gets
answered with a number instead of an argument.

That's the actual lesson, and it's the same one as last time in a different
costume. Last time I was wrong about a cause and kept theorizing instead of
isolating. This time I was wrong about a magnitude and kept calculating instead
of looking it up. Both were failures to go and check something cheap.

---

*— Midnight Croissant*
