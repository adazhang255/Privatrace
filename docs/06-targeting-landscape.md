# 6. The targeting landscape — does this project age well?

**This document contains a correction.** In an earlier conversation I told you that ad
platforms were shifting toward on-device targeting via Privacy Sandbox / the Topics API,
that less of the interesting activity would be visible as network requests over time, and
that this should make you cautious about investing in tracker detection.

**That conclusion was wrong, and the actual outcome is close to the opposite.** Privacy
Sandbox has been cancelled. Third-party cookies are staying in Chrome. Network-visible
tracking is not going away. Tracker detection is a *more* durable investment than I
implied, not less.

Here's the full picture, since the reasoning matters more than the verdict.

## Level 1 — what happened, briefly

**Privacy Sandbox** was Google's multi-year programme to remove third-party cookies from
Chrome and replace them with privacy-preserving APIs. The **Topics API** was its
interest-targeting piece: instead of trackers following you across sites, *Chrome itself*
would observe your browsing, derive a handful of coarse interest topics on-device, and hand
only those topics to advertisers. Related APIs covered auctions (Protected Audience,
formerly FLEDGE) and conversion measurement (Attribution Reporting).

If that had shipped, it would genuinely have undercut this project: the interesting
inference would have moved inside the browser, where no extension could watch it.

It didn't ship. The sequence:

- **July 2024** — Google retreats from outright cookie deprecation to a "user choice"
  model: a prompt letting people decide.
- **April 2025** — Google confirms it will not launch that prompt either. Third-party
  cookies stay in Chrome, managed through existing privacy settings.
- **October 2025** — Google announces it is retiring most Privacy Sandbox APIs, Topics
  among them. Deprecation in Chrome 144 (January 2026), removal targeted for Chrome 150
  (July 2026). UK CMA oversight of the commitments winds down.

The stated reasons: low adoption by the ad industry, continued antitrust pressure, and —
once Google decided to keep third-party cookies — no remaining need for a replacement.

*Verify current status before making a decision on it. This is a fast-moving area and the
dates above are near the edge of what I can confirm.*

## Level 2 — what this means for PrivaTrace

### The reassuring part

**Your core mechanisms remain valid for the foreseeable future.**

- Third-party cookies still exist in Chrome, so `chrome.cookies.onChanged` keeps showing
  real cross-site tracking.
- Tracker requests still happen over the network, so `chrome.webRequest` observation keeps
  working.
- The ad-settings pages still exist — Google's My Ad Center, Meta's ad preferences — so the
  disclosed panel still has a source.
- Fingerprinting is, if anything, *more* relevant: with no cookie deprecation forcing
  change, and with Safari and Firefox already blocking third-party cookies by default,
  fingerprinting remains the durable workaround for a tracker that wants cross-site
  identity anyway.

There is no looming platform change that invalidates the design in `architecture.md`.
Invest in tracker detection on its merits.

### The part that was right for the wrong reason

The advice I gave — *the disclosed vs. inferred comparison stays meaningful either way* —
happens to still hold, and it's worth understanding why, because it's the real strategic
insight.

The disclosed/inferred comparison doesn't depend on any particular tracking *mechanism*.
It asks: what does the platform claim to know, and does your own activity support it? That
question survives cookies, Topics, fingerprinting, server-side tagging, or whatever comes
next, because platforms will keep publishing interest profiles (regulation increasingly
requires it) and users will keep being unable to explain them.

**Mechanism-watching dates. The comparison doesn't.** That's the durable core, and it's
the part you've already built.

## Level 3 — what *actually* erodes your visibility

Privacy Sandbox was the wrong thing to worry about. These are the real ones, and none of
them are hypothetical:

**Server-side tagging.** Instead of the page calling `google-analytics.com`, it calls
`metrics.theirsite.com` — their own server — which forwards the data onward. From the
browser this is indistinguishable from an ordinary first-party request. You see one benign
call; the tracking still happens, entirely out of your view. This is already widely
deployed and growing, and it's the single biggest limit on any browser-side observer.

**CNAME cloaking.** A tracker's domain disguised as a first-party subdomain via DNS.
Extensions can't see DNS resolution, so you can't unmask it. Covered in
[doc 2](02-observing-behaviour.md).

**First-party data and data clean rooms.** Increasingly, targeting is built from data the
platform already holds (purchases, logged-in behaviour, uploaded customer lists) and
matched server-to-server. Nothing crosses the browser at all. An email address handed to an
advertiser offline is invisible to every extension ever written.

**In-app tracking.** Substantial browsing happens in mobile apps, where there's no
extension model at all.

The honest summary: **a browser extension sees a shrinking share of total tracking, but
what it sees isn't going away.** Your UI should be careful not to imply completeness — "here
is what this page did in your browser" is defensible; "here is everyone who tracked you" is
not. That framing discipline is the same one your synthesis bar already applies to
correlation, and it should extend here.

## A separate note: the feature you're underselling

If what you were asking about was the *product* concern rather than the platform one — the
`?`-flagged interests — here it is in more depth, because I think it's the most valuable
unbuilt thing in the project.

Right now `demo.html` hardcodes one flag: Google claims "Baby products," and it's marked
with a dashed border and a `?`. It's a static string.

**With the inference engine in place, that gap is now computable.** You have two profiles:
what the platform discloses, and what your own activity supports. Three cases fall out:

| Case | Meaning | Value to user |
|---|---|---|
| Platform says it, activity supports it | Expected | Low — mildly reassuring |
| Activity supports it, platform doesn't say it | They may know and not disclose it | Interesting |
| **Platform says it, activity supports nothing** | **Unexplained inference** | **This is the whole product** |

That third row is the moment the abstraction becomes personal. Not "advertisers track you"
— everyone's heard that and it changes nothing — but "*Google thinks you're shopping for
baby products, and nothing you did in the last fortnight explains why.*" That's specific,
it's about them, and it prompts the obvious question: where did that come from? Offline
data? A household member? An inference from a correlation you can't see? You can't answer
it — and **being unable to answer it is the finding**, not a limitation.

You already have `disclosedMatch()` computing the overlap. The unmatched set is just its
complement. Rendering that as a first-class panel — "3 interests we can't explain" — is a
small amount of code for the project's strongest moment.

One caution, consistent with the rest of your UI discipline: an unexplained interest is
*unexplained*, not *wrong*. Your activity log is partial (it only covers the retention
window, and only this browser) so absence of evidence is genuinely weak evidence here. Say
"your recorded activity doesn't explain this," not "this is inaccurate."

## Sources

- [Google Pulls The Plug On Topics, PAAPI And Other Major Privacy Sandbox APIs — AdExchanger](https://www.adexchanger.com/privacy/google-pulls-the-plug-on-topics-paapi-and-other-major-privacy-sandbox-apis-as-the-cma-says-cheerio/)
- [Privacy Sandbox — Wikipedia](https://en.wikipedia.org/wiki/Privacy_Sandbox)
- [Google Privacy Sandbox officially shuts down — Usercentrics](https://usercentrics.com/knowledge-hub/what-is-google-privacy-sandbox/)
- [After years of uncertainty, Google says it won't be deprecating third-party cookies in Chrome — Digiday](https://digiday.com/marketing/after-years-of-uncertainty-google-says-it-wont-be-deprecating-third-party-cookies-in-chrome/)
- [Google's changing approach to third-party cookies — Usercentrics](https://usercentrics.com/knowledge-hub/google-third-party-cookies/)
- [The Rise and Fall of Google's Privacy Sandbox (arXiv)](https://arxiv.org/html/2607.00693v1)
