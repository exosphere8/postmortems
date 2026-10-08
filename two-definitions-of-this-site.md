# Two definitions of "this site"

*A post-mortem from [BlockThem](https://github.com/exosphere8/BlockThem), my Manifest V3 ad
blocker, in which three files had to agree on what "this site" means, and only two of them did.*

---

## The setup

BlockThem has a "Pause on this site" button. A pause has to land in three places:

- **background.js** turns the allowlist into one `allowAllRequests` rule for
  `declarativeNetRequest`.
- **content.js** skips injecting the CSS that hides leftover ad slots.
- **popup.js** decides whether the button says "Pause" or "Resume", and edits the list.

Before release, each answered "is this site paused?" like this:

```js
// background.js: Chrome does the matching
condition: { requestDomains: allowlist, resourceTypes: ['main_frame', 'sub_frame'] }

// content.js
allowlist.some(d => host === d || host.endsWith('.' + d))

// popup.js
const paused = allowlist.includes(host);
```

You'll spot it in five seconds. I wrote all three and didn't.

## Why it looked fine

Honestly, I was a bit smug about content.js. I'd thought about subdomains there. I even
remembered the dot in `'.' + d`, so `notgithub.com` doesn't count as `github.com`. Subdomains:
handled.

In one file.

The popup only ever *writes* exact hostnames, so it *read* exact hostnames too. Add `host`,
check for `host`, remove `host`. Symmetric. And on the page you paused from, it works perfectly,
which is exactly the page you'd check it on.

## What was actually happening

Chrome's docs for `requestDomains` cover it in one line: "Sub-domains of the listed domains are
also matched." So the rule meant "this domain and everything under it", and content.js agreed.
The popup meant "this exact hostname". Two definitions of "this site", and the file with the
button got outvoted.

Pause on `github.com`, then open `docs.github.com`:

1. The page is fully paused. Chrome lets every request through and content.js hides nothing.
2. The popup checks `includes('docs.github.com')`. False. It offers **Pause**.
3. Click it. `docs.github.com` gets added, which changes nothing. Already paused.
4. Next time it offers **Resume**. Click it. It removes `docs.github.com`, as written.
5. `github.com` still covers the page, so it stays paused, and the popup goes back to offering
   **Pause** as if blocking were on.

Round and round. From that page, nothing in the popup could turn blocking back on.

The popup is the humblest file in the project: a number, a switch, one button. It tried so
hard. It was just looking for the wrong name on the list.

Nobody got stuck in that loop. Before publishing v1 on 8 Oct, I reviewed it from three angles
(MV3 API correctness, user-visible behavior, filter quality) and had every finding challenged by
a second pass whose only job was to refute it. Ten findings, four survived. This one was found
twice, independently, and survived both times.

## The fix

The popup now asks the question the other two ask:

```js
// before
const paused = allowlist.includes(host);
hostEl.textContent = host;
// on click:
const next = paused ? allowlist.filter(d => d !== host) : [...allowlist, host];

// after
const covers = d => host === d || host.endsWith('.' + d);
const paused = allowlist.some(covers);
hostEl.textContent = allowlist.find(covers) || host;
// on click:
const next = paused ? allowlist.filter(d => !covers(d)) : [...allowlist, host];
```

The catch: Resume on `docs.github.com` now removes `github.com`, so the parent and all its other
subdomains resume too. On purpose. A plain `requestDomains` list can't say "github.com, except
docs". `excludedRequestDomains` could, but that needs a list of exceptions all three files
understand, and v1 doesn't have one. So the line under the button now names the entry doing the
pausing, and on that page it says `github.com`. Less clever. More honest, I think.

And yes, the predicate still lives in two files, each with a comment pointing at the network
rule. Better than two opinions. Not as good as one.

## What I'd take from this

- **When three components must agree on a definition, write it once.** Or at least run all
  three against the same cases. `github.com` on the list, `docs.github.com` in the tab: that
  one case would have caught this.
- **If the browser is one of the three, it wins.** Chrome had already defined "this site" in
  `requestDomains`. My JavaScript doesn't get a vote. It gets to copy.
- **Test from the page next door.** The page you paused from is guaranteed to pass. The bug
  lives one subdomain over.
