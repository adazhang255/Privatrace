# PrivaTrace

**A browser extension that shows you what advertisers know about you — and where they got it.**

Ad platforms already have a page where you can see your interest profile. Google has one,
Meta has one, Amazon has one. Almost nobody looks at them, because they're buried in
settings, they're on three different sites, and they tell you *what* the platform thinks
about you but never *why*.

PrivaTrace pulls all three into one dashboard, watches what trackers actually do on the
pages you visit, and — this is the part the platforms don't give you — builds its own
guess at your ad profile where **every single line can be traced back to the thing you did
that caused it.**

> **Status: prototype.** `demo.html` is a working front-end mockup with simulated data.
> The extension described in `architecture.md` has not been built yet. See
> [What's real and what isn't](#whats-real-and-what-isnt).

---

## Try it in 10 seconds

```sh
open demo.html
```

That's it — it's a single file with no build step and no dependencies.

Then:

1. **Session audit** (the default view) — click between the fake site tabs at the top and
   watch the tracker log fill up on the right.
2. **Your data & profile** — click the second nav button. This is the interesting part.
3. Hit **Download JSON** and open the file. Everything PrivaTrace holds about you, in one place.

---

## The three things it shows you

### 1. Disclosed — what the platforms admit to

The left column of the audit view. These are the interest tags Google, Meta and Amazon
list on their *own* ad-settings pages. PrivaTrace reads them off the page while you have
it open and keeps a copy so you can see all three side by side.

You'll notice tags marked with a dashed red border and a `?`. Those are ones PrivaTrace
couldn't connect to anything you did — the "why on earth does Google think I want baby
products" category.

### 2. Observed — what actually happened on the page

The right column. Every cookie written, every request to a known tracker, every
fingerprinting API call, live as it happens. This is the raw behaviour, not anyone's
summary of it.

### 3. Inferred — PrivaTrace's own guess, with receipts

The **Your data & profile** view. PrivaTrace runs its own small recommendation algorithm
over your activity and tells you what it *would* target you with, in a sentence you can
check:

> **Cars & automotive** — *high confidence*
> You would likely be shown ads about **cars**, because on Sep 4 and Sep 11 you looked at
> cars content on **wheelsandtorque.com**.

Expand the card and you get every visit that fed into it, with dates.

**Why bother building a second guess when the platforms give you theirs?** Because a
guess you can audit is the only way to judge one you can't. If PrivaTrace can work out
"this person is car shopping" from twelve rows of history, that tells you something about
what a company with a thousand times more data can do. And when the platform lists an
interest that *your own* activity doesn't support — that gap is the story.

Cards outlined in blue carry a badge like `Google also lists "Travel"` — that's where
PrivaTrace's guess and the platform's stated profile agree.

---

## Your data is temporary, and you can delete any of it

The **Your data & profile** view opens with a panel titled *"What's being held right now."*
It's first on the page on purpose — you see what's held before you see what was concluded
from it.

PrivaTrace needs your browsing activity to explain anything. That makes it a tool that
would otherwise assemble the most revealing dataset a person owns, in order to complain
that other people assemble it. The way out is not to hold it:

- **Nothing persists.** The activity log is an in-memory array. No localStorage, no
  database, no disk. Close the tab and it's gone, with no recovery path — deliberately, because
  a recovery path is a stored copy.
- **Entries expire on their own.** Default 14 days, selectable between 3 and 30. Expired
  rows are *deleted, not summarised* — a "was interested in cars last month" rollup would
  be smaller than the rows behind it but no less sensitive, so there isn't one.
- **You can delete any slice of it:** a single entry, one whole interest category
  ("Forget this"), all activity, or everything including the synced platform profiles.
  Because it's partitioned by topic, forgetting one interest doesn't cost you the rest.
- **Nothing derived is cached.** Every score, confidence level and sentence is recomputed
  from the activity log on each render. That's what makes deletion mean deletion — a cached
  score would leave a conclusion standing after its evidence was gone, which is exactly the
  failure mode this tool exists to criticise.

The honest tradeoff: a longer window would make better inferences. The bet is that making
the point with two weeks of data is more convincing than needing a year of your history to
make it.

## Download your data

Same idea as Instagram's "Download Your Information", but it takes a second instead of 48
hours, and the file is assembled in your browser rather than on someone's server.

| Format | What's in it | Good for |
|---|---|---|
| **JSON** | Everything — disclosed interests, full activity log, inferred profile with evidence, current session stats | Feeding into other tools, keeping a dated snapshot |
| **Readable report** | The same data as a formatted HTML page | Actually reading it; sending to someone |
| **Activity CSV** | Just the activity log | Spreadsheets, charts |

The export is built with `Blob` + an object URL and saved straight to your downloads
folder. There is no server in this project, so there's nothing for it to upload to.

Because nothing else persists, **an export is the only way to make any of this durable** —
and then it's an ordinary file you own and can delete like any other. That asymmetry is the
design: persisting your data should take a deliberate action, while forgetting it should be
the default. That's the inverse of every platform whose profile this tool displays.

---

## How the inference works

The scoring is deliberately simple, because the whole point is that you can check it.

```
score = (distinct_days × 2 + total_visits) × recency_multiplier

recency_multiplier:  last 7 days → 1.0
                     8–21 days   → 0.6
                     older       → 0.3
```

Confidence is a plain rule on top:

| Confidence | When |
|---|---|
| **high** | 3+ separate days of activity, most recent within 3 weeks |
| **medium** | 2+ separate days |
| **low** | a single day |

**Distinct days matter more than raw visit count** — reading eight car articles in one
sitting is one afternoon of curiosity, whereas checking back on three different days is
someone actually shopping. Doubling the day count encodes that.

**Old activity decays but never disappears.** Something you researched two months ago is
weaker evidence than yesterday, but it isn't nothing, so the multiplier bottoms out at 0.3
instead of dropping to zero.

**Expired activity can't reach a conclusion.** `inferProfile()` prunes past the retention
window before it scores anything, so a row that has aged out contributes to nothing.

**Nothing is shown that can't cite its own evidence.** `inferProfile()` returns the source
rows alongside every score, and the UI renders straight from them. There is no code path
that produces a claim without the receipts attached — that's enforced by the data
structure, not by discipline.

This is intentionally *not* a good targeting algorithm. It's a legible one. A real ad
system uses collaborative filtering across millions of users, and its output can't be
explained this way even by the people who built it. That contrast is the lesson.

---

## What's real and what isn't

| | Status |
|---|---|
| Dashboard UI, all three views | ✅ Working in `demo.html` |
| Inference engine + evidence trail | ✅ Working, real logic on synthetic data |
| Data export (JSON / HTML / CSV) | ✅ Working, real files |
| Retention window, expiry, granular delete | ✅ Working, real behaviour |
| Activity log | ⚠️ Seeded with synthetic history; live tab-visits append to it |
| Tracker/cookie/fingerprint log | ⚠️ Simulated on a timer, not real interception |
| Reading real platform ad settings | ❌ Not built — designed in `architecture.md` |
| Actual browser extension | ❌ Not built |

The synthetic data is there so the UI can be evaluated before the hard part is built.
Every simulated element is labelled as such in the interface.

---

## The files

| File | What it is |
|---|---|
| `README.md` | This file |
| `demo.html` | The working prototype — open it directly, no build step |
| `architecture.md` | Design for the real extension: manifest, content scripts, extractors, storage, inference, retention |
| **[`docs/`](docs/README.md)** | **Background reading — what to learn before building, high level down to the depth needed** |
| `wireframe.md` | ASCII layout sketch the UI was built from |

### The background docs

If you're picking this up in order to build it, start at [`docs/README.md`](docs/README.md).
Seven rundowns covering Manifest V3, observing page behaviour, content scripts and isolated
worlds, DOM extraction, ad-tech vocabulary, whether the project ages well, and the
ephemeral-data design. Each starts at "what is this and why do I care" and stops at the
depth this project actually needs.

---

## If you're building this for real

The prototype is the easy half. Read `architecture.md` before starting — especially
§9, on terms of service. The short version of the hard parts:

1. **The extractors will break.** Reading interest tags off Google/Meta/Amazon settings
   pages means depending on undocumented DOM structure that changes without notice.
   Budget for maintenance, and make sure a broken extractor never overwrites good stored
   data with an empty result (`architecture.md` §3).
2. **Keep the activity log ephemeral.** This is now the prototype's design and it should
   survive into the real build: `chrome.storage.session` (cleared on browser close), never
   `chrome.storage.local`, and never `chrome.storage.sync`, which leaves the device. MV3's
   non-persistent service worker — normally an obstacle — happens to be aligned with you
   here. See [docs/07](docs/07-ephemeral-data-design.md), including the list of
   perfectly-reasonable-sounding requests that would break this.
3. **Correlation is not causation, and the UI has to keep saying so.** When a tracker
   fires on a travel site and Google lists "Travel", that's two separately-collected
   signals that happen to line up. It is *not* proof the one caused the other. The
   synthesis bar says this explicitly and it should stay that way.
4. **Read the terms of service.** Reading your own settings page with a content script is
   at the low-risk end of automated access, but most platforms' ToS language doesn't
   cleanly distinguish it from scraping. Fine for personal use; get a real opinion before
   distributing. Note that the Takeout / Download-Your-Information import path sidesteps
   the question entirely — one more reason to build it first
   ([docs/04](docs/04-dom-extraction.md)).
5. **Build out the unexplained interests.** The `?`-flagged interest in the demo is a
   hardcoded string, but with the inference engine in place the gap is now computable:
   platform claims it, your activity supports nothing. "3 interests we can't explain" is a
   small amount of code for the project's strongest moment — see
   [docs/06](docs/06-targeting-landscape.md).

---

## License

Not yet specified.
