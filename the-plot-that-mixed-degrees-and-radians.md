# The plot that mixed degrees and radians

*A build post-mortem from [balancing-sculpture](https://github.com/exosphere8/balancing-sculpture):
a viral "physics-defying" sculpture, explained with a Python model, a Manim animation and a
live web simulation. Nothing here was hard. Almost everything still went wrong once.*

---

## The setup

A reel showed a steel figure standing on one toe on a small plate, arms swept back to two
heavy cylinders on long curved rods. It wobbles and never falls. The physics is short: the
weights pull the combined centre of mass *below* the foot, so the sculpture is a physical
pendulum hanging from its toe.

I wanted three things built from one source of truth:

- `physics.py` — point masses, centre of mass, moment of inertia, period, and a `solve_ivp`
  simulation.
- `scene.py` — a Manim explainer that uses those numbers.
- `web/` — a page with a draggable simulation, sliders, and the derivation.

With masses estimated from the video (figure 160 g, weights 240 g and 200 g), the model gives
the centre of mass 14.4 cm below the foot, I = 0.0323 kg·m², and a period of 1.23 s. Those
numbers survived. The tooling around them took most of the detours.

## Detour one: Manim would not install

`pip install manim` failed while building `manimpango`:

```
RequiredDependencyException: pangocairo >= 1.30.0 is required
```

That one is honest — Manim's text rendering needs the Pango development headers, and the
error says so. Installing `libpango1.0-dev` fixed it and revealed the next failure, which
was not honest at all:

```
ERROR: Failed building wheel for srt
AttributeError: install_layout. Did you mean: 'install_platlib'?
```

`srt` is a single-file subtitle library. Nothing about it should fail to build. My first
reaction was to blame `srt`, then to blame setuptools versions, and I tried building it
with `--no-build-isolation` after upgrading setuptools. Same error.

The `install_layout` attribute is not part of upstream setuptools. It comes from the
distribution's patched Python packaging, which expects a Debian-style install layout. Building
a legacy `setup.py` package against the system interpreter walks into that patch.

**The fix** was the same one I wrote about in
[the numpy post-mortem](numpy-was-never-the-problem.md): stop installing into the system
Python. A fresh virtual environment with an upgraded pip and setuptools installed Manim
0.22 on the first try.

Then the first `MathTex` failed because `latex` was installed but `dvisvgm` was not. Manim
renders LaTeX to DVI and converts it to SVG; having the first half of that pipeline doesn't
help. One more package.

## Detour two: the first render was correct and unreadable

The first preview rendered without errors and looked wrong. The figure — the thing the whole
video is about — was a thin squiggle next to two bold cylinders, and the "centre of mass"
label sat directly on top of the right-hand weight.

Nothing was miscalculated. I had drawn the figure at the model's true scale with the same
stroke widths as the thin arms, so it vanished. And the label was placed with `next_to(...,
RIGHT)`, which is right next to whatever happens to be on the right.

I only caught it because I pulled seven frames out of the video and looked at them as a
contact sheet. "It rendered" and "it reads" are separate checks.

## Detour three: the plot that mixed degrees and radians

The web page plots θ(t). The sculpture swings about ±15°, so the y-axis should auto-scale to
±15°. It showed ±90°, with the trace squashed into a flat ripple in the middle.

The scaling line was:

```js
const maxAbs = Math.max(10, ...history.map((h) => Math.abs(h.th))) * 180 / Math.PI;
```

The intent was "the larger of 10° and the biggest swing, in degrees". What it does is compare
`10` (meant as degrees) with angles in *radians* (about 0.26), pick the 10, and then convert
that 10 from radians to degrees: 573°. The axis is clamped at 90°, so 90° it was.

Every individual piece of that line is right. The bug is that a bare number had no unit, and
it got compared with a number that had a different one. The fix converts first, then compares:

```js
const maxAbs = Math.max(10, ...history.map((h) => Math.abs(h.th) * 180 / Math.PI));
```

## Detour four: the Python model quietly assumes balance

The web page has an "Unbalanced" preset with a heavier left weight. It should not stand
upright; it should settle at the angle where the centre of mass is straight under the toe.
To get that, the page integrates the general torque:

```
I θ'' = −M g (x_c cos θ − y_c sin θ) − c θ'
```

While writing that, I noticed the Python model doesn't. `physics.py` uses the textbook form
`I θ'' = −M g d sin θ`, which is only correct when the torques already cancel (Σ m x = 0). For
the sculpture's own numbers they do, so every output is right. But the moment someone feeds
it a lopsided configuration, it will report a perfectly upright pendulum that the real object
would never be.

I've left this as a documented limitation for now, with a test that at least flags the
lopsided case as having a tipping moment. The web simulation is the general one.

## Detour five: "created" is not "exists"

At the end I had two commits ready and a repo that I believed existed. The push target came back as
repository not found. I had started creating it on GitHub but never actually submitted the
form. Opening the URL showed a 404, which settled it in five seconds after I'd spent longer
than that assuming it was a permissions problem.

Second, smaller one: the first commits used `exosphere8@users.noreply.github.com`. The noreply
address GitHub actually assigns this account includes the numeric ID
(`101312106+exosphere8@...`), and that is the form that links commits to the profile. I rewrote the
author on both commits and force-pushed, which was safe only because the repo was minutes old.

Third: the Pages deploy failed on the first push because Pages wasn't enabled yet when the
workflow ran. Enabling it and re-running the workflow fixed it. The failure was correct; it
was just early.

## What was actually going on

None of these were physics problems. The model was right from the first run. Every detour was
at a boundary — between my code and the system's Python, between "renders" and "readable",
between degrees and radians, between the special case and the general one, between "I did
the step" and "the thing exists".

## The takeaway

**Convert at the boundary, and give every bare number a unit in its name.** If `maxAbs` had
been called `maxAbsDeg` and the history field `thRad`, the bad comparison would have looked
wrong the moment I typed it. The same habit covers the others: check the thing itself (the
frame, the repo URL, the deploy status), not my memory of having made it.
