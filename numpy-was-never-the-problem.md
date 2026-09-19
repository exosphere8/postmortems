# The numpy version was never the problem

*A debugging post-mortem from a Kaggle notebook, in which I blame three different
versions of the same library before noticing that the version was irrelevant.*

---

## The setup

I was standing up an environment for a 3D+time cell-tracking competition. The
requirements were unremarkable: `zarr` to read the image volumes, `scipy` for
spatial queries, and `tracksdata` — a graph library that isn't on PyPI, so it
installs from a git URL.

```bash
pip install -q "zarr>=3.0.10" scipy "tracksdata @ git+https://github.com/royerlab/tracksdata@main"
```

The install succeeded. The import did not.

## Crash one

```
ImportError: cannot import name '_center' from 'numpy._core.umath'
```

The traceback was long and went somewhere odd. My code imported `tracksdata`,
which imported `scipy.spatial`, which — several layers down — reached
`scipy._lib.array_api_compat.numpy`, which does `from numpy import *`. That
wildcard import pulled in `numpy.strings`, a submodule almost nobody imports
directly, and `numpy/_core/strings.py` tried to import `_center` from
`numpy._core.umath`. That function wasn't there.

Note what's strange about this. Both files are *inside numpy*. One part of numpy
was asking another part of numpy for something it didn't have. That should be
impossible in a coherent installation, and I noticed it was strange, and then I
ignored what it was telling me.

## Hypothesis one, and why it was wrong

pip had resolved numpy to 2.5.3. My conclusion: 2.5.3 is a bad release. Pin
below it.

```bash
pip install -q --force-reinstall --no-deps "numpy<2.4" scipy
```

This is where I introduced a second bug while chasing the first. `--no-deps`
tells pip to skip dependency resolution entirely — which means it no longer
checks whether the numpy and scipy I just forced together are actually
compatible. I had removed the safety rail specifically so I could force through
a combination nobody had validated.

## Crash two

```
AttributeError: module 'numpy._core._multiarray_umath' has no attribute '_blas_supports_fpe'
```

A different missing name, in a different numpy internal module, reached through
a different path (`numpy.testing` this time, again dragged in by that wildcard
import).

I read this as confirmation of hypothesis one — *see, versions matter* — and
formed hypothesis two: my `--no-deps` hack had paired an old numpy with a scipy
built against a newer one. Reasonable. Also wrong.

## Crash three, and the thing that finally landed

Fresh container. No forcing, no `--no-deps`. I excluded only the specific
version I'd decided was cursed and let pip's resolver do its job properly:

```bash
pip install -q "numpy!=2.5.3" "zarr>=3.0.10" scipy "tracksdata @ git+..."
```

It resolved cleanly to numpy 2.0.2 and scipy 1.16.3 — a combination pip itself
vouched for. Then:

```
AttributeError: module 'numpy._core._multiarray_umath' has no attribute '_blas_supports_fpe'
```

The identical failure. Third numpy version, no forcing, resolver satisfied.

That's the moment the actual signal was unmissable: **when the same bug survives
three different versions of the thing you're blaming, you are blaming the wrong
thing.** Every hypothesis I'd formed was a hypothesis about *which version*. The
evidence had been saying "not a version problem" since crash one, in the form of
numpy contradicting itself internally, and I'd spent three cycles refusing to
hear it.

## What was actually happening

The Kaggle base image ships numpy preinstalled. When pip "upgrades" numpy in
that environment, it has to replace an existing installation — remove the old
files, write the new ones. On this image, that replacement wasn't clean. The
pure-Python files got updated; some compiled extension files did not.

The result is a single directory called `numpy` containing files from two
different builds. `numpy/_core/strings.py` from one version, expecting
`_center`. `numpy/_core/_multiarray_umath.so` from another, not providing it.
The package is internally inconsistent, and the specific symptom depends on
which files happened to land — which is exactly why each attempt produced a
different missing name.

This also explains why the crashes always surfaced through `numpy.testing` and
`numpy.strings`, submodules that normal code never touches. Only scipy's
`from numpy import *` reached deep enough to step on the mismatched parts.
Ordinary `import numpy` worked fine and printed a perfectly reasonable version
number, which made the installation look healthy right up until it wasn't.

## The fix

Stop upgrading in place. If you never overwrite the existing installation, there
are no mixed files.

Install into an isolated location instead and put it first on the import path:

```bash
pip install --target=/kaggle/working/pylibs zarr scipy tracksdata
```

```python
import sys
sys.path.insert(0, "/kaggle/working/pylibs")
```

A fresh directory receives one complete, self-consistent copy of each package.
The system numpy is untouched, so nothing it half-replaces can break. A virtual
environment achieves the same thing by the same mechanism; `--target` is just
the lighter-weight version, and it avoids needing `venv` to work at all — which,
as it happens, mattered.

## The proof

The isolated install resolved numpy to **2.5.3** — the exact version I had spent
the evening convinced was broken — and it imported perfectly.

That's the whole case, in one line of output. The version was never the problem.
The *installation method* was.

## Two other things that cost me time

**`python -m venv` silently produced a venv with no pip.** The `bin/` directory
had `python`, `python3`, `python3.12` and nothing else:

```
Error: Command '[... '-m', 'ensurepip', '--upgrade', '--default-pip']' returned non-zero exit status 1.
```

`ensurepip` is the stdlib module `venv` uses to bootstrap pip into a new
environment, and it was broken on this image. The workaround is to skip it
entirely: `virtualenv` ships its own pip wheel and never calls `ensurepip`.

```bash
pip install virtualenv && python -m virtualenv --system-site-packages /path/to/venv
```

**`pip download` is not the same as `pip wheel`.** The competition disables
internet access for scored submissions, so everything had to be installed from
local files. My first attempt used `pip download`, then failed offline with:

```
pip subprocess to install build dependencies did not run successfully
```

`pip download` saves whatever the index offers, which for a git-sourced package
is *source*, not a built wheel. Installing source means building it, and
building means fetching build tools — from the internet I no longer had.
`pip wheel` does the building up front and writes finished `.whl` files, so the
offline install has nothing left to compile:

```bash
pip wheel -w ./wheels "zarr>=3.0.10" scipy "tracksdata @ git+..."
pip install --no-index --find-links=./wheels --target=./pylibs zarr scipy tracksdata
```

## What I'd take from this

The technical lesson is narrow and useful: in a prebuilt image full of packages
you didn't install, prefer adding an isolated copy over upgrading in place.
Upgrades have to succeed at deletion as well as writing, and deletion is where
the quiet failures live.

The debugging lesson is broader and cost me more. I had a genuinely strange clue
in the first thirty seconds — one part of a library failing to find another part
of itself — and I explained it away three times because I'd already decided what
kind of bug this was. Each new error, arriving with a different missing symbol,
felt like fresh evidence for the version theory. It was actually the same
evidence, repeated, against it.

The tell for this failure mode is when your fixes keep producing *different*
symptoms rather than progress. Different symptoms from the same root cause look
deceptively like progress. They aren't. They're the system telling you your
model of the problem is wrong, in the only language it has.
