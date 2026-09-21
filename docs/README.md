# PrivaTrace — background reading

Written for: you, building this project. These are learning rundowns, not API reference —
each one starts at "what is this and why do I care" and goes down to the depth this
specific project actually needs, then tells you where it stops.

## Read in this order

| # | Doc | Why it's here | Depth needed |
|---|---|---|---|
| 1 | [Browser extensions & MV3](01-extensions-mv3.md) | The platform everything runs on. Most likely to surprise you. | Deep — you'll fight this |
| 2 | [Observing page behaviour](02-observing-behaviour.md) | Makes the "Observed" panel possible or impossible | Deep |
| 3 | [Content scripts & isolated worlds](03-content-scripts-isolated-worlds.md) | Required for both extraction and fingerprint detection | Deep — the fiddliest part |
| 4 | [Extracting data from pages you don't control](04-dom-extraction.md) | Your "Disclosed" panel, and the project's main fragility | Moderate, but it's a *practice* not a fact |
| 5 | [Ad-tech vocabulary](05-adtech-vocabulary.md) | So your UI labels are accurate | Shallow — vocabulary only |
| 6 | [The targeting landscape](06-targeting-landscape.md) | Whether this project ages well. **Contains a correction.** | Moderate — strategic, not technical |
| 7 | [Designing for ephemeral data](07-ephemeral-data-design.md) | The principle behind the code that now exists | Deep — it's your differentiator |

## The one-paragraph version

You're building a Manifest V3 Chrome extension. Its hard parts are, in order: reading
interest tags off three companies' undocumented settings pages (permanently fragile —
mitigate by preferring the official data-export path), detecting fingerprinting (needs
code injected into the page's own JS world, not the extension's), and holding a browsing
history responsibly (solved by not holding one — see doc 7). Everything else is ordinary
front-end work you've already done in `demo.html`.

## What you can skip

- **Machine learning.** Your algorithm is a scoring rule and should stay one. Legibility
  is the product.
- **MV2 tutorials.** Most extension content online predates MV3 and teaches patterns that
  no longer work. Check the date on everything.
- **Building your own tracker list.** Ship EasyPrivacy or DuckDuckGo Tracker Radar.
- **Server-side anything.** This project has no server, deliberately, and shouldn't grow one.
