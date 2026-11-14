# pip freeze put pytest in my image

14 Nov 2026 · [KhanalChess](https://github.com/exosphere8/KhanalChess) ·
fixed in PR #18, merged 2026-11-14 at 23:57 (+0545)

## The setup

KhanalChess is a FastAPI service packaged as a Docker image. Its
`requirements.txt` was first written with `pip freeze`, which lists every
package in the virtual environment, whatever each one is for. The Dockerfile
installs `requirements.txt`, so everything in that file shipped in the image:
pytest and its helpers included.

## The wrong turns

**1. "pip freeze gives me a correct requirements file."**
It does, in one sense: it reproduces the environment exactly. [FILL IN: why
this looked right to me at the time.] What it cannot know is which packages
the app needs to run and which only my tools need.

**2. "I will just freeze again when I add psycopg."**
Next I needed the PostgreSQL driver. Running `pip freeze` again would have
written ruff into `requirements.txt` as well, because ruff was installed in
the same venv for the lint job. Instead I appended `psycopg==3.3.9` and
`psycopg-binary==3.3.9` by hand and left ruff and tzdata out. That was a
stopgap, not a fix, and I noted it as one.

**3. "I can tell which packages are dev tools by their names."**
Some names are obvious (pytest, ruff). Some are not. `opentelemetry-api` is
not something I installed on purpose. A guess would either have broken the
image or left test tools in it. So I stopped guessing and asked pip.

## What was actually happening

`pip show <package>` prints two lines that settle the question: `Requires`
(what it needs) and `Required-by` (what needs it). Following those lines:

- `opentelemetry-api` is needed by `fastapi`, so it is a runtime package.
- `colorama`, `packaging` and `truststore` were only there for the dev tools.

The result is two files:

- `requirements.txt`: the 18 packages the API needs to run.
- `requirements-dev.txt`: starts with `-r requirements.txt`, then adds
  colorama (with the marker `sys_platform == "win32"`), httpcore2, httpx2,
  iniconfig, packaging, pluggy, Pygments, pytest, ruff and truststore.

Because the dev file includes the runtime file, the two cannot drift apart.
`tzdata` is in neither file: psycopg pulls it in on Windows by itself. The
Dockerfile did not change at all. It already installed only
`requirements.txt`. The Dockerfile was right and the file was wrong.

CI changed in one place: the test job installs `requirements-dev.txt`, and
`cache-dependency-path` lists both files.

**Evidence, in both directions:**

- Nothing missing: a fresh throwaway venv, `pip install -r
  requirements-dev.txt`, and 11 tests passed. My main venv could not prove
  this, because it still held the old packages.
- Nothing extra: I rebuilt the image (57.2 s, the pip layer reran). `pip
  list` inside it showed the 18 runtime packages plus pip, and no pytest,
  httpx2, ruff, truststore, colorama or pluggy.

## The takeaway

Never write `pip freeze > requirements.txt`. Sort packages with `pip show`
(Requires and Required-by), and prove a requirements file in both directions:
a fresh venv shows nothing is missing, and `pip list` inside the image shows
nothing is extra.
