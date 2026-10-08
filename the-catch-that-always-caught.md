# The catch block that always caught

*A post-mortem from [BlockThem](https://github.com/exosphere8/BlockThem), a Manifest V3 ad
blocker, in which a try/catch had one job and the catch did it every single time.*

---

## The setup

BlockThem blocks ads in two layers. `declarativeNetRequest` (DNR) stops requests to ad
networks, and `content.js` injects a stylesheet that hides the empty boxes left behind
(`.ad-container`, `[data-ad-slot]` and friends). The content script runs in iframes too
(`all_frames: true`).

"Pause on this site" adds the site to an `allowAllRequests` rule, and the content script
checks the same allowlist before hiding anything.

Iframes got about as much thought as you'd expect. One on a paused site should count as
paused, so I judged it by the top-level site:

```js
let host;
try { host = window.top.location.hostname; }
catch { host = location.hostname; }
```

Ask for the top page's host. If the browser says no, use your own. Defensive. Tidy. I was
pleased with myself.

Before publishing v1 on 8 October, I reviewed it from three angles (MV3 API correctness,
user-visible behavior, filter quality) and had every finding challenged by a second pass
whose only job was to refute it. This one survived. It never shipped, so nobody hit it.

## Why it looked right

In the top frame it works perfectly, because `window.top` is just `window`. And a try/catch
reads like a story about odds: usually this works, occasionally it won't. I'd filed the
catch under *occasionally*.

Wrong drawer.

## What was actually happening

A content script gets the same cross-origin restrictions as its frame. In a cross-origin
iframe, reading `window.top.location.hostname` throws a `SecurityError`, the catch runs, and
you get the iframe's own host.

The read only succeeds when the iframe is same-origin with the top page, and then the top's
hostname *is* its own.

So the try branch never changed the answer. Not once. The fallback was the whole program
and my clever line was decoration.

That was my catch block: the quiet line nobody looked at, doing all the work.

The cost: when the pause rule matches the top-level page, Chromium allowlists the whole
frame tree, so a paused site's iframes load their ads. But the content script in each
cross-origin iframe saw its own, unpaused host and hid whatever matched the generic ad
selectors anyway. That can break an embedded player that waits on its ad slot, which is
exactly the kind of thing you press Pause to fix.

## The first fix was wrong too

The obvious repair: Chromium exposes `location.ancestorOrigins` to a frame even across
origins, so judge by its last entry, the top-level origin. One review angle proposed just
that. Looked right to me.

The refuting pass confirmed the bug and rejected the fix. The pause rule also matches
`sub_frame`, so a paused domain's iframes are allowlisted *on other sites too*. Pause a video
site, open an unpaused page that embeds it, and DNR lets that iframe's requests through.
Judge it by the top host and it hides its ad slots again. Same bug, mirrored.

(Should a pause reach embeds on other sites at all? A separate finding raised that and was
rejected as a product decision. Fine. Then the content script has to agree.)

## The fix

Copy what DNR does instead of my mental model of it: skip the hiding CSS if this frame
**or any ancestor** is on a paused site.

```js
// before: try { host = window.top.location.hostname } catch { host = location.hostname }

// after
const hosts = [location.hostname];
for (const origin of Array.from(location.ancestorOrigins || [])) {
  try { hosts.push(new URL(origin).hostname); } catch {}
}
const paused = hosts.some(h => allowlist.some(d => h === d || h.endsWith('.' + d)));
```

Yes, there's still a catch in there. I see it. This one only catches an opaque `"null"`
origin (a sandboxed frame, say), which has no hostname to check. A paused ancestor that
shows up as `"null"` won't count. Small gap, and this time I know it's there.

## What I'd take from this

- **A fallback that returns a plausible value can hide that the main path never runs.** Mine
  returned a real hostname every time.
- **Test the branch you think is rare.** One page with a cross-origin iframe would have shown
  the catch firing.
- **Make fallbacks loud while building.** One `console.debug` in that catch and I'd have
  seen it.
- **When you mirror another system's rules, copy its rules.** DNR asks "is this frame, or
  anything above it, allowlisted?" I was asking "what's the top site?" Those agree often
  enough that I never noticed they're different questions.
