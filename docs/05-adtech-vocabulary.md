# 5. Ad-tech vocabulary

Shallow by design. You need this so your UI labels are *accurate* — a privacy tool that
misuses these terms loses credibility with exactly the audience that would care about it.
You don't need the industry's business mechanics.

## The core distinction: first-party vs third-party

**First-party** — the site you're on, setting a cookie for its own use. `cart_id` on a shop
is first-party. Mostly benign; it's how logins and shopping carts work.

**Third-party** — a different domain, embedded in the page, setting a cookie readable
across every other site that also embeds it. That cross-site readability is the entire
mechanism of behavioural tracking: it's what lets one company recognise you on a travel
site and then on a news site and connect the two.

Precision matters here: the distinction is about **the domain the cookie belongs to**, not
who wrote the code. A tracker's script running on a first-party domain (see CNAME cloaking
in [doc 2](02-observing-behaviour.md)) sets a *first-party* cookie while doing third-party
work. This is why your "first-party" label should describe the request, not imply safety.

## Terms your UI will touch

**Pixel / tracking pixel** — a tiny (classically 1×1 transparent) image request whose only
purpose is the *request itself*. Loading it tells the server you saw the page, and carries
cookies and a referrer. No JS required, which is why they work in email too. In
`chrome.webRequest`, a tracker domain with `type: 'image'` is very likely a pixel — a
detail your UI can surface meaningfully.

**Tag / container** — a snippet site owners paste in (Google Tag Manager being the big
one) that then loads other trackers dynamically. Relevant to you because it means **the
set of trackers on a page isn't static** and isn't knowable from the HTML — it's assembled
at runtime. Good justification for observing rather than analysing source.

**Cookie syncing (cookie matching / ID syncing)** — two ad companies comparing notes so
that A's ID for you and B's ID for you are known to be the same person. Shows up as
redirect chains between ad domains carrying IDs in the URL. This is the mechanism that
turns many small trackers into one large profile, and it's a genuinely good thing to
surface if you can detect the chains, because it's the least intuitive part of the system.

**Fingerprinting** — identifying a browser by its configuration (canvas rendering quirks,
fonts, audio stack, screen, hardware) instead of a stored ID. Matters because it's
*stateless*: clearing cookies doesn't help, and there's nothing for the user to delete.
See [doc 3](03-content-scripts-isolated-worlds.md) for detection, and its warning about
overclaiming.

**Retargeting** — being shown ads for a thing you already looked at. The most *visible*
form of tracking, and the one your users will already have noticed. Useful as an on-ramp
in explanatory copy.

**Audience / segment** — the advertiser-side name for the interest buckets your "Disclosed"
panel shows. "In-market for: Automotive" is a segment.

**Inference** — the platform deciding something about you that you never stated. This is
precisely what your project is about, and worth using the word deliberately: the difference
between *declared* data (you typed it) and *inferred* data (they guessed) is the heart of
why an ad profile feels invasive.

**DSP / SSP / ad exchange** — demand side, supply side, and the marketplace between them.
You will see these domains in your logs. You don't need the business model; just know
they're marketplace infrastructure, not the advertiser.

**RTB (real-time bidding)** — the auction, run in the ~100ms while the page loads, in which
your profile data is broadcast to bidders. Worth knowing because it explains something
counterintuitive and alarming: your data is sent to *many* companies that don't win the
auction and therefore never show you an ad. The "how many companies saw this?" framing is
much stronger than "how many ads did you see?"

## Tracker lists

A "known tracker list" is just a curated domain list, maintained by humans and imperfect.
Two you'd actually ship:

- **EasyPrivacy** — the tracker-focused list behind uBlock Origin et al. Filter-rule
  syntax; powerful, complex to parse fully.
- **DuckDuckGo Tracker Radar** — structured JSON, with **domain → owning company**
  mapping and categories. Better fit for you, as argued in
  [doc 2](02-observing-behaviour.md), because company attribution makes far better UI
  copy than bare hostnames.

Be aware and be honest: these lists have false positives and false negatives, and
"unclassified" is a real and common state — which your demo already displays rather than
hiding. Keep that.

## Where to stop

You don't need: header bidding, prebid.js, viewability metrics, attribution windows, CPM
economics, identity resolution vendors, or clean rooms. If your UI never says a word, you
don't need to learn it.
