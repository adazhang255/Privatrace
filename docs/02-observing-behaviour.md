# 2. Observing page behaviour

This is the "Observed on this page" panel: every cookie written, every tracker contacted,
every fingerprinting attempt. Right now it's a timer emitting fake rows. This is what it
takes to make it real.

## Level 1 — the high level

Three separate signals, three completely different mechanisms. They feel like one feature
in the UI, which is misleading — treat them as three sub-projects of very different
difficulty.

| Signal | Mechanism | Difficulty |
|---|---|---|
| Network requests | `chrome.webRequest` (observation only) | Easy |
| Cookies | `chrome.cookies` + its change event | Easy |
| localStorage writes | Page-world injection | Hard — see [doc 3](03-content-scripts-isolated-worlds.md) |
| Fingerprinting | Page-world injection | Hard — see [doc 3](03-content-scripts-isolated-worlds.md) |

The good news is that the two easy ones carry most of the value. "This page contacted 4
known trackers" is the line that lands with people, and it's the cheapest thing here.

## Level 2 — webRequest vs declarativeNetRequest

This is the distinction I flagged as critical, and here's the actual resolution:

MV3 removed the **blocking** form of `chrome.webRequest` for most extensions. Under MV2 an
ad blocker could register a listener that inspected each request and returned a verdict to
cancel it. That's gone — blocking is now done declaratively via
**`declarativeNetRequest`**, where you register static rules and the browser applies them
without consulting your JS. The point was performance and privacy: the browser no longer
has to hand every URL you visit to extension code.

**`chrome.webRequest` still exists in MV3 for observation.** The read-only events —
`onBeforeRequest`, `onCompleted`, `onErrorOccurred` and friends — work fine. What you
cannot do is return a value that changes the request's fate.

**PrivaTrace only ever observes.** So:

> Use `chrome.webRequest` in observe-only mode. `declarativeNetRequest` is for blockers
> and is not your tool. You don't need it at all.

This is worth being clear-eyed about because a lot of writing from the MV3 transition is
ad-blocker-community commentary framing `webRequest` as "killed." For your purposes it
isn't. Verify the current API surface in Chrome's docs before you build, but the design is
sound.

```js
// observation only — no return value, no "blocking" in extraInfoSpec
chrome.webRequest.onBeforeRequest.addListener(
  (details) => {
    // details.url, details.initiator, details.type ('script', 'image', 'xmlhttprequest'…)
    // details.tabId lets you attribute it to the right tab
    recordRequest(details);
  },
  { urls: ["<all_urls>"] }
);
```

Note `details.type` — it's genuinely useful for your UI. A tracker loaded as a
1×1 `image` is a classic pixel; the same host as `xmlhttprequest` is something more
active. You can surface that distinction.

## Level 3 — the depth you need

### Classifying a request as a tracker

You need a list, and you should not write one. Two real options:

- **EasyPrivacy** — the tracker-focused companion to EasyList, the filter list behind
  uBlock/AdBlock. Text-based filter syntax; you'll need a parser, and full filter syntax
  is genuinely complex (domain anchors, options, exceptions). For your purposes you can
  cheat: extract just the plain domain rules and ignore the rest.
- **DuckDuckGo Tracker Radar** — a JSON dataset of tracker domains with owner companies
  and categories. **Much better fit for you.** It's structured data rather than filter
  syntax, and critically it maps domains to *parent companies*, which lets your UI say
  "contacted 3 domains owned by Meta" instead of listing opaque hostnames. That's a far
  better product.

Either way, ship the list in the package and update it on releases. Don't fetch it at
runtime — that's a network call revealing usage, in a privacy tool.

Your three-way classification in `demo.html` (tracker / first-party / unclassified) is
good and honest. Keep "unclassified" visible rather than silently assuming unknown means
safe.

### First-party detection is subtler than it looks

Comparing `details.initiator` to the request hostname gets you most of the way, but:

- CDNs are first-party in spirit, different domain in fact (`cdn.example.com` — fine, same
  eTLD+1 — but also `examplecdn.net`, which isn't).
- You want **eTLD+1** comparison, not naive string matching, or `evil-example.com.attacker.net`
  fools you. Use the Public Suffix List; don't hand-roll domain parsing.
- **CNAME-cloaked trackers** are deliberately disguised as first-party subdomains
  (`analytics.example.com` resolving to a tracker). Extensions largely can't see through
  this, since DNS resolution isn't exposed. Accept the limitation and don't claim
  completeness.

### Cookies

`chrome.cookies.onChanged` gives you writes, including ones set via HTTP `Set-Cookie`
headers that a content script could never see. The `cookie.domain` field with a leading dot
and the `httpOnly` / `sameSite` flags are all there. This is the easiest real win in the
whole project — do it early for the morale boost.

### Attribution and lifecycle

Two things that will cause bugs:

**Tab attribution.** `details.tabId` is `-1` for requests not tied to a tab (service
workers, prefetch). Your per-domain panel needs to drop or bucket those, not crash.

**When to reset.** Your demo clears counts on tab switch. Real navigation is messier: SPAs
change URL without a page load, `onCommitted` vs `onHistoryStateUpdated` are different
events, and a background tab keeps making requests while you're not looking. Decide
explicitly whether the panel shows "this page load" or "this tab, this session" and make
the label match. Getting this wrong makes your numbers look fabricated.

### Volume

A single ad-heavy news page can fire hundreds of requests. Your log caps at 40 rows, which
is right for display, but don't accumulate unbounded arrays in memory behind it — and given
doc 7's direction, don't accumulate them at all beyond what's on screen.

## Where to stop

You don't need: request modification, `declarativeNetRequest` rule authoring, header
inspection beyond `Set-Cookie`, or the devtools protocol.
