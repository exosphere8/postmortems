# I blocked Hotjar on hotjar.com

*A post-mortem from [BlockThem](https://github.com/exosphere8/BlockThem), my Manifest V3 ad
blocker, in which a list of tracker domains was quietly also a list of websites to break.*

---

## The setup

All of BlockThem's network blocking lives in one file, `rules.json`, enforced by Chrome's
`declarativeNetRequest` engine. In v1 that file held a single rule: 56 ad and tracker domains,
a `block` action, and `main_frame` excluded, so typing `criteo.com` into the address bar would
still get you a page.

I was smug about it. No regexes, no URL paths. Nothing clever, so nothing clever to get
wrong.

Before publishing, I reviewed it from three angles (MV3 API correctness, user-visible
behavior, filter quality) and had every finding challenged by a second pass whose only job was
to refute it. Two of the three angles found this bug on their own. Neither refutation pass
could kill it, and both came back with a catch I'd missed.

## The assumption

`requestDomains: ["hotjar.com"]` means "block Hotjar." That was my entire mental model.

It survived because it's right almost everywhere. Hotjar's script on some online shop is a
stranger at the party and should be shown the door. Subdomain matching even felt like a gift:
no need to list `static.hotjar.com`, `script.hotjar.com` and whatever they call the next one.

## What was actually happening

`requestDomains` matches the domain and every subdomain. Without a `domainType`, it doesn't
care who's asking. Third-party, first-party, the vendor's own page: on the list means gone.

Picture a Hotjar customer opening `insights.hotjar.com`. The HTML loads, because I excluded
`main_frame`. Then the page asks its own domain for scripts and API data, and BlockThem says
no. To all of it. Same on `app.adroll.com`, `app.mouseflow.com`, `marketing.criteo.com`, and
the Taboola and Outbrain advertiser consoles. People do their jobs in those, and I'd have
broken every one.

The badge would have counted each of those requests as blocked, too. A proud little red number
on a page that doesn't work.

My README even promises that top-level page loads are never blocked, "so you can still visit
these sites directly." Technically true! You could visit. You just couldn't do anything once
you got there.

The EasyPrivacy filter list has this exact entry: `||hotjar.com^$third-party`. I'd never asked
what that suffix was for. This is what it's for.

Review caught it before release, so nobody stared at a dead dashboard. Somebody
would have.

## The obvious fix, and the catch

Add `"domainType": "thirdParty"`. Chrome compares each request with the page or frame that
made it, by registrable domain (eTLD+1). So `insights.hotjar.com` fetching from
`static.hotjar.com` is first-party and goes through, while Hotjar on anyone else's site stays
blocked. Done.

Nope.

Four entries weren't vendor domains. They were subdomains of giant platforms:
`adservice.google.com`, `ads.linkedin.com`, `bat.bing.com`, `ads.yahoo.com`. On
`linkedin.com`, a request to `px.ads.linkedin.com` *is* first-party. Same registrable domain.
A blanket `thirdParty` would have quietly stopped blocking those four hosts on their parent
sites. No error, no broken page, just less blocking than I thought I had. I'd have shipped it
feeling great about myself.

## The fix

Split it in two. Before:

```json
{ "id": 1, "priority": 1, "action": { "type": "block" },
  "condition": {
    "requestDomains": [ "doubleclick.net", "...all 56..." ],
    "excludedResourceTypes": ["main_frame"] } }
```

After:

```json
{ "id": 1, "priority": 1, "action": { "type": "block" },
  "condition": {
    "requestDomains": [ "doubleclick.net", "...52 vendor domains..." ],
    "domainType": "thirdParty",
    "excludedResourceTypes": ["main_frame"] } },
{ "id": 2, "priority": 1, "action": { "type": "block" },
  "condition": {
    "requestDomains": [ "adservice.google.com", "ads.linkedin.com",
                        "bat.bing.com", "ads.yahoo.com" ],
    "excludedResourceTypes": ["main_frame"] } }
```

Vendors get blocked when they follow you onto other people's sites and left alone on their
own. The four platform hosts stay blocked everywhere, same as before. The one thing I gave up
is blocking Hotjar on hotjar.com, which was the point.

An ad blocker should have small ambitions, the "I Wanna Be Yours" kind from
[the first post](saved-is-not-applied.md). Be boring and useful, and never be the reason
someone's dashboard is blank.

## What I'd take from this

- **A blocklist entry is a claim about context.** "Block X" is half a sentence. The other half
  is "...when loaded by whom?"
- **`requestDomains` with no `domainType` means everywhere, including X's own site.** If you
  don't mean that, say so.
- **Before fixing a whole list, look for the entries that don't fit.** `thirdParty`
  was right for 52 domains and quietly wrong for 4.
