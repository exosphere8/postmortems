# Six of my ten bug reports were wrong

*A post-mortem of the review I ran on [BlockThem](https://github.com/exosphere8/BlockThem)
before releasing it, in which most of what I found wasn't there.*

---

## The setup

BlockThem is a small Manifest V3 ad blocker for Chromium browsers. Before shipping v1 on
8 Oct 2026, I reviewed it from three angles: MV3 API correctness, user-visible behavior, and
filter quality. Then I had every finding challenged by a second pass whose only job was to
refute it.

Fourteen findings came back. Four bugs had been caught from two angles each, so: ten
distinct ones. Ten bugs in a few hundred lines of code. I was, briefly, very pleased with how
thorough I'd been.

Briefly.

## The assumption

A finding with a line number, a concrete scenario and a ready-to-paste patch is a bug.

All ten had that. The analytics one cited Google's documented callback patterns, pointed at
uBlock Origin's surrogate scripts, and arrived with stand-ins for `analytics.js` and `gtm.js`
already written. It was convincing. It was also wrong.

Honestly, I wanted them all to be real. Every finding turned up eager and sincere, full
"I Wanna Be Yours" energy, asking only for a yes. I wanted to say yes to all ten.
But wanting a finding to be real is just wanting to be right, and neither is evidence.

## What was actually happening

Six of the ten died under refutation, for good reasons:

- **"Blocking Google Analytics and Tag Manager breaks checkouts that wait for the analytics
  callback."** Only on sites that won't navigate until a callback fires and set no timeout.
  Google's own GTM docs recommend pairing `eventCallback` with `eventTimeout` for exactly
  that reason. Also, blocking trackers is the job, and "Pause on this site" exists.
- **"Overlapping `sync()` calls collide on dynamic rule id 1."** That needs the allowlist to
  go from empty to non-empty while a second storage write lands within roughly 100 ms, cold
  start included. The only things that write settings are one Pause click or one flip of the
  switch at a time, so nothing in the code could get there.
- **"Major ad networks are missing."** A wish list, not a defect. The manifest says
  "common", not "all".
- **"Including `sub_frame` in the allow rule turns off blocking in that site's embeds
  everywhere."** Accurate mechanics, deliberate scope. Adblock Plus's `$document`
  allowlisting works the same way.
- **"Other tabs stay stale until reload"** is a known limitation, and **"the grey disabled
  badge never appears"** contradicted itself: its own description admitted the grey shows on
  tabs that still have a count.

The one that stung was `sub_frame`. Its fix and the fix for a real bug (ad hiding inside
cross-origin iframes on paused sites) rewrite the same check in `content.js` in opposite
directions. Mix them and network blocking and ad hiding disagree again, which is the exact
thing the real fix was for. Fixing everything wouldn't have been thorough. It would have been
a regression with good intentions.

The survivors got pushback too. [The reload race](saved-is-not-applied.md), caught from two angles independently, was
confirmed and then corrected: Chromium decides `allowAllRequests` when the navigation
*commits*, not when the request starts. Its severity dropped from high to medium. Two other
confirmed findings had their proposed patches rewritten, because as written they'd have
caused new problems.

An odd pattern, from a sample far too small to be a rule: every survivor had been raised
from two angles. Every casualty, from one.

## The fix

The rule-id collision died as a finding, but its code went anyway. Fixing the reload race
added a second trigger for `sync()`, which is exactly the overlap that finding needed. With
the old read-then-write still there, the first Pause click could have hit a duplicate-id
error and never reloaded the tab.

```js
// before (simplified): read what's there, then replace it
const existing = await chrome.declarativeNetRequest.getDynamicRules();
await chrome.declarativeNetRequest.updateDynamicRules({
  removeRuleIds: existing.map(r => r.id),
  addRules: allowlist.length ? [allowRule] : []
});

// after: one fixed id, removed unconditionally, every sync queued
await chrome.declarativeNetRequest.updateDynamicRules({
  removeRuleIds: [ALLOW_RULE_ID], // ids that don't exist are ignored
  addRules: allowlist.length ? [allowRule] : []
});
```

Rejected isn't ignored. It means "not a bug in *this* code". Change the code, ask again.

## What I'd take from this

- **A review that only confirms isn't a review.** It's a to-do list with a confident tone.
- **Ask something to argue back.** A refutation pass is cheap. Acting on a plausible-but-wrong
  finding isn't: new files, new rules, sometimes a fix that breaks another fix.
- **Review the patch, not just the finding.** Real bugs came with fixes that would have made
  new ones.
- **Accurate isn't the same as a bug.** The `sub_frame` finding described Chromium perfectly
  and was still a design choice.
