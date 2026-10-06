# The partial crawl that hid a price drop

*A post-mortem from [BookWatch](https://github.com/exosphere8/bookwatch), a price tracker,
in which "compare with the previous run" quietly destroyed the information it was supposed
to report.*

---

## The setup

BookWatch crawls a catalogue, stores every scrape as a *run*, and tells you what changed:
price drops and rises, restocks, sell-outs, new products, delisted products. The 2.0 rewrite
gave runs a proper lifecycle. A run is marked `ok` only when the crawl finishes, it records
whether it reached the last page (`complete`), and reports read successful runs only.

The question "what changed?" had an obvious answer, and I implemented it:

```python
before = storage.snapshot(conn, old.id)   # the previous successful run
after = storage.snapshot(conn, new.id)    # the latest successful run
```

Diff the two. A product in `after` but not in `before` is new, unless it has been seen in
some earlier run, in which case skip it. A product in `before` but not in `after` was
delisted, but only if the new run crawled the whole catalogue, because a partial crawl
can't tell "removed" from "not reached".

The tests passed. Every case I had thought of was covered.

## Why it looked right

With full crawls, "the previous run" and "everything I know" are the same thing. Every run
sees every product, so diffing consecutive runs is exactly the right comparison.

The trouble is that BookWatch doesn't promise full crawls. `scrape --pages 3` was the
default, and partial runs were a first-class feature. I had handled partial runs on one side
of the comparison (no "delisted" from a partial crawl) and not on the other.

## What was actually happening

Three runs:

| Run | Crawl | What it saw |
|---|---|---|
| 1 | full | A, B and Y, all at £10 |
| 2 | partial (`--pages 1`) | only A |
| 3 | full | A at £10, B at **£5 and sold out**, Y gone |

Comparing run 3 with run 2:

- **B** is in run 3 but not in run 2. It has been seen before, so it is not new, and the code
  `continue`s. The 50% drop and the sell-out are never compared with anything.
- **Y** is in neither run 2 nor run 3, so there is nothing to call "removed".

Comparing run 4 (another full crawl, nothing changes) with run 3: B is £5 in both, and Y is
absent in both. Nothing to report.

The drop wasn't reported late. It was never reported at all. The comparison window
determined which facts could ever become changes, and a single short crawl was enough to
make one fall through the gap permanently.

## The fix

The baseline was the wrong concept. The right question per product is not "what did the
previous run see?" but "what did I last know about *this product*?"

So `compare()` now takes each product in the new run and compares it with that product's own
most recent observation in *any* earlier successful run:

```sql
-- the latest ok observation of each product before the run being reported
WHERE o.run_id = (
    SELECT MAX(o2.run_id) FROM observations o2 JOIN runs r ON r.id = o2.run_id
    WHERE o2.product_id = o.product_id AND r.status = 'ok' AND o2.run_id < :before
)
```

Delisting needed the same idea from the other side. A product missing from a complete
crawl is reported as removed only if no complete crawl since its last sighting has reported
it already:

```python
already_reported = last_complete is not None and last_complete > prev["run_id"]
```

Every change is now reported exactly once, by the first run that is able to see it.

## The proof

The scenario above is now a test, and it is the one I would point to if someone asked
whether the change detection can be trusted:

```python
def test_changes_hidden_by_a_partial_crawl_are_still_caught(conn):
    ok_run(conn, [book("A"), book("B", 1000), book("Y")])
    ok_run(conn, [book("A")], complete=False)
    ok_run(conn, [book("A"), book("B", 500, in_stock=False)])

    _, _, changes = diff.latest_changes(conn)
    assert kinds(changes) == [("price_drop", "B"), ("sold_out", "B"), ("removed", "Y")]
```

A second test pins the other half: a delisting is reported by the first complete crawl and
then never again.

## Two other things that were wrong

**I let floats back in one step after banning them.** Prices are stored as integer cents
precisely because `0.29 * 100` is `28.999999999999996`. Then the `--min-pct` filter computed
the move as a float percentage. A rise from 100 to 129 cents is 29%, which became
`28.999999999999996`, so `changes --min-pct 29` dropped it. The fix keeps the comparison in
exact arithmetic:

```python
Decimal(abs(new - old)) * 100 >= Decimal(str(min_pct)) * old
```

**On Windows, NUL claims to be a terminal.** BookWatch writes UTF-8 when its output is
redirected and keeps the console's encoding when you are looking at a console. It decided
which case applied with `isatty()`. The `NUL` device is a character device, so
`bookwatch history ... > NUL` reported a TTY, kept the ANSI code page, and crashed on the
sparkline characters. The check now asks whether the stream is a real Windows console
object. This one shipped, which is why there is a 2.0.1.

## What I'd take from this

- **A baseline is a claim about what you know.** If the baseline can be incomplete, compare
  against knowledge (the last observation of each thing), not against the last observation
  of everything.
- **Handle a special case on both sides of a comparison, or on neither.** I had made partial
  runs safe for "removed" and unsafe for every other kind of change.
- **Precision decisions leak at the boundaries.** Storing exact values means nothing if the
  comparison that matters is done in floats.
