# Saved is not applied

*A post-mortem from [BlockThem](https://github.com/exosphere8/BlockThem), a Manifest V3 ad
blocker, in which "Pause on this site" reloaded the page before the pause existed.*

---

## The setup

BlockThem has two layers. Network blocking is Chromium's `declarativeNetRequest`: a static
blocklist, plus one dynamic `allowAllRequests` rule for the sites you've paused. Cosmetic
hiding is a content script that injects CSS to hide leftover ad slots, unless storage says
the site is paused.

In the version I was about to ship, "Pause on this site" did this:

```js
await chrome.storage.local.set({ allowlist: next });
chrome.tabs.reload(tab.id);
```

Save, reload, done. Two lines. The README even promised it "takes effect right away". I was
smug about it.

## Why it looked right

Storage was the source of truth. The `await` resolved, so the truth was written, so the
reload would see it.

And half the extension *did* see it. The content script runs at `document_start` and reads
storage itself, so it always got the new allowlist. One layer agreeing with me made the whole
thing feel correct.

## What was actually happening

Storage doesn't block anything. The network rules do, and they live somewhere else.

The service worker rewrote them on `storage.onChanged`. If it had dozed off, as idle MV3
workers do, it had to start up first. Then `sync()` made four calls in a row: `storage.get`,
`updateEnabledRulesets`, `getDynamicRules`, and `updateDynamicRules`, which indexes the new
rules and writes them to disk.

The popup waited for none of it.

Which would just be slow, except for one thing. Chromium decides whether a frame is allow-all
once, when its navigation commits (`OnDidFinishNavigation` in `ruleset_matcher_base.cc`).
`updateDynamicRules` swaps in a fresh matcher with no record of any frame, and nothing
rechecks frames that already committed. If the reload commits first, the allow rule never
applies to that page. Not late. Never, for the life of that document.

So "Pause on this site" got you the worst of both:

- network blocking still on, because the rules didn't know yet;
- hiding CSS off, because storage did.

Ads still blocked, empty ad boxes on show, and two halves of one extension arguing about
whether the site is paused. Resume and the on/off switch got off lighter: those changes apply
to every request after they land, so only the reload's earliest requests were wrong.

There's a song on *AM*, Arctic Monkeys' "I Wanna Be Yours", where the narrator offers to be
any dull, useful thing around someone's house. That's an ad blocker's whole job: do the chore
when asked, no fuss. Mine said yes, then did the chore after you'd already looked.

Nobody hit this. Before release I reviewed the extension from three angles and had every
finding challenged by a second pass whose only job was to refute it. Two of the three caught
it independently. A lot of attention for two lines.

## The fix

The popup now asks the background to apply the change, and reloads only when it hears back:

```js
await chrome.storage.local.set(settings);
await chrome.runtime.sendMessage({ type: 'sync' });   // new: wait for the rules
await chrome.tabs.reload(tabId);
```

The background replies only once its sync has finished.

The fix had its own trap, and a *rejected* finding pointed at it.

One review had flagged that two overlapping syncs could both get an empty list from
`getDynamicRules`, both add rule id 1, and one would fail as a duplicate. The refuting pass
threw it out: nothing in the code at the time could realistically make two syncs collide.
Then it added a note.

My fix can.

One click now fires `storage.onChanged` *and* the message. With nothing paused yet, both
syncs could read that empty list, and if the message's sync was the one that failed, no
answer came back. No answer, no reload.

So syncs now go through a promise queue, one at a time, and the allow rule is removed by its
fixed id:

```js
removeRuleIds: [ALLOW_RULE_ID],   // was: whatever getDynamicRules() had just returned
```

Chrome ignores ids that don't exist, so it's safe to send every time.

## What I'd take from this

- **"Saved" is not "applied".** If one part stores a setting and another acts on it, wait
  for the one that acts.
- **Find out when your change gets read.** Checked per request, a race costs a few requests.
  Decided at commit, it costs the whole page.
- **A rejected bug can come back through your fix.** A second trigger turned a race nothing
  could start into one a single click could.
