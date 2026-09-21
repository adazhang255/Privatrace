# PrivaTrace — disclosed-data architecture (manual / passive)

Scope decision: the extension never auto-navigates to a platform's settings page and never scrapes in the background without you having opened that page yourself. Every capture is either (a) you clicking "open settings" from the dashboard, which opens a normal foreground tab, or (b) you visiting that page anyway in the course of using the platform. Either way, capture happens passively once the tab is open — no headless requests, no scheduled background scraping.

This trades automation for lower risk: narrower permissions, no bot-like traffic pattern, and a request flow that's indistinguishable from you browsing normally.

---

## 1. Component map

```
extension/
├── manifest.json
├── background/
│   └── service-worker.js       # message hub, storage writes, "open settings" trigger
├── content-scripts/
│   └── disclosed-capture.js    # runs only on matched settings-page origins
├── sidepanel/
│   ├── panel.html / panel.js   # dashboard UI (disclosed + observed panels)
├── lib/
│   ├── extractors/
│   │   ├── google.js
│   │   ├── meta.js
│   │   └── amazon.js
│   └── schema.js                # normalization + validation
```

Only three things are new for the manual/passive design versus a generic "auditor" build: the `content-scripts/disclosed-capture.js` gate, the `extractors/` folder, and the "open settings" button wiring in the panel. Everything else (observed-panel logic, storage, general UI) is unchanged from the earlier prototype.

---

## 2. manifest.json — narrow permissions, not `<all_urls>`

```jsonc
{
  "manifest_version": 3,
  "host_permissions": [
    "https://myadcenter.google.com/*",
    "https://accountscenter.facebook.com/*",
    "https://www.amazon.com/adprefs*"
  ],
  "content_scripts": [
    {
      "matches": [
        "https://myadcenter.google.com/*",
        "https://accountscenter.facebook.com/*",
        "https://www.amazon.com/adprefs*"
      ],
      "js": ["content-scripts/disclosed-capture.js"],
      "run_at": "document_idle"
    }
  ],
  "permissions": ["storage", "tabs", "sidePanel"],
  "background": { "service_worker": "background/service-worker.js" }
}
```

Scoping `host_permissions` to exactly these three origins (instead of `<all_urls>`) matters for two reasons: it makes the install-time permission prompt honest about what the extension actually touches, and it means the content script physically cannot run anywhere except these settings pages — there's no code path for it to fire elsewhere even if something went wrong.

---

## 3. Content script — passive extraction, not a scheduled job

These settings pages are client-rendered SPAs, so the interest list usually isn't in the DOM at `document_idle` — it loads a beat later via JS. The script waits for it instead of parsing immediately.

```js
// content-scripts/disclosed-capture.js
(function () {
  const extractor = pickExtractor(location.hostname); // see lib/extractors/
  if (!extractor) return;

  const observer = new MutationObserver(() => {
    const result = tryExtract(extractor);
    if (result.status === 'ok') {
      observer.disconnect();
      chrome.runtime.sendMessage({
        type: 'disclosed_capture',
        platform: extractor.platform,
        interests: result.interests,
        capturedAt: Date.now(),
      });
    }
  });

  observer.observe(document.body, { childList: true, subtree: true });

  // give up after 15s rather than observing forever
  setTimeout(() => observer.disconnect(), 15000);
})();

function tryExtract(extractor) {
  try {
    const interests = extractor.extract(document);
    if (!interests.length) return { status: 'empty' };
    return { status: 'ok', interests };
  } catch (e) {
    return { status: 'error', error: String(e) };
  }
}
```

**Never overwrite good data with an empty result.** If `status` isn't `'ok'`, the background worker leaves the existing stored profile untouched and just marks it as "sync attempt failed" — a redesigned page shouldn't silently blank out yesterday's real data.

---

## 4. Extractors — one per platform, isolated and swappable

The selectors below are **illustrative placeholders**, not verified against the live DOM. Real values have to come from inspecting each page directly (DevTools → Elements) since none of this is documented anywhere, and they will need re-checking periodically as each platform ships redesigns.

```js
// lib/extractors/google.js
export default {
  platform: 'google',
  extract(doc) {
    const chips = doc.querySelectorAll('[data-testid="interest-chip"]');
    return Array.from(chips).map(el => ({
      label: el.textContent.trim(),
      confidence: el.hasAttribute('data-uncertain-match') ? 'low' : 'high',
    }));
  },
};
```

```js
// lib/extractors/meta.js
export default {
  platform: 'meta',
  extract(doc) {
    const rows = doc.querySelectorAll('[role="listitem"][data-ad-topic]');
    return Array.from(rows).map(el => ({
      label: el.getAttribute('data-ad-topic'),
      confidence: 'high',
    }));
  },
};
```

```js
// lib/extractors/amazon.js
export default {
  platform: 'amazon',
  extract(doc) {
    const items = doc.querySelectorAll('.ad-preference-category');
    return Array.from(items).map(el => ({
      label: el.textContent.trim(),
      confidence: 'high',
    }));
  },
};
```

Keeping each extractor to a single `extract(doc) → interests[]` function means a broken selector on one platform can't take down capture on the other two, and swapping in a corrected selector after a redesign is a one-file change.

---

## 5. Normalized schema

Every extractor output gets passed through this shape before storage, regardless of source platform:

```ts
interface DisclosedInterest {
  id: string;                                  // hash(platform + label)
  platform: 'google' | 'meta' | 'amazon';
  label: string;                                // raw text from the page
  confidence: 'high' | 'low' | 'unknown';
  capturedAt: number;                           // epoch ms
  sourceType: 'scrape';                         // reserved for future 'export' path
}
```

`sourceType` is already in the schema even though this version only has one value, so adding the Takeout/Download-Your-Information import path later doesn't require a schema migration — just a second producer writing into the same shape.

---

## 6. Background worker — storage + the manual trigger

```js
// background/service-worker.js
chrome.runtime.onMessage.addListener((msg, sender) => {
  if (msg.type === 'disclosed_capture') {
    const key = `disclosed:${msg.platform}`;
    chrome.storage.local.set({
      [key]: {
        interests: msg.interests,
        capturedAt: msg.capturedAt,
        lastError: null,
      },
    });
  }
  if (msg.type === 'open_settings') {
    chrome.tabs.create({ url: SETTINGS_URLS[msg.platform] });
    // capture happens passively once that tab's content script fires —
    // no further action taken here
  }
});

const SETTINGS_URLS = {
  google: 'https://myadcenter.google.com',
  meta: 'https://accountscenter.facebook.com/ad_preferences',
  amazon: 'https://www.amazon.com/adprefs',
};
```

`open_settings` is the entire "manual" half of manual/passive: it opens a normal tab and does nothing else. The content script on that tab does the actual work once it loads, on its own, with no further messages from the background worker telling it to run.

---

## 7. Freshness, not expiry

Data is never deleted for being old — it's just labeled:

| Age since `capturedAt` | Dashboard state |
|---|---|
| < 48h | "synced Xh ago" |
| 48h – 14d | "synced X days ago" |
| > 14d | "stale — last synced X days ago", chips shown dimmed |
| never captured | "not yet connected" |

Staleness is a nudge to re-sync, not a reason to hide data you already have — a two-week-old profile is still more informative than nothing.

---

## 8. What this version deliberately doesn't do

- No background `chrome.tabs.create` fired automatically or on a schedule — every tab-open is a direct result of a click in the dashboard.
- No data-export (Takeout / Download Your Information) parsing yet — schema supports it, nothing produces it.
- No cross-device sync of the captured profile — `chrome.storage.local` is per-browser-install by design here.
- No retry/backoff logic beyond the 15s MutationObserver timeout — a failed capture just sits as "sync failed" until you manually retry.

## 9. Standing risk to flag before shipping beyond personal use

Reading your own account settings page with a content script is different, legally, from an unattended scraper — but most platforms' terms of service use broad "automated access" language that doesn't cleanly distinguish the two. This design (user-initiated tab open, no scheduling, no headless requests) is the lower-risk end of the spectrum, but it isn't a legal opinion. Worth a real ToS read, or legal review, before distributing this to anyone beyond yourself.

---

## 10. Activity log and the inference engine

The disclosed panel answers "what does Google think about me". It cannot answer "why", because the platform doesn't say. The inference engine exists to make that second question answerable at all — by building a parallel profile from data the user *can* inspect, and requiring every claim in it to carry its own evidence.

### 10.1 Activity schema

```ts
interface ActivityEntry {
  id: string;        // local, for targeted deletion
  ts: number;        // epoch ms — when it happened
  host: string;      // the site
  topic: string;     // key into the TOPICS table
  detail: string;    // human-readable: 'read "EV charging costs explained"'
}
```

`detail` is deliberately a display string rather than structured fields. It only ever gets shown to the user or written to their export; nothing computes on it. Keeping it unstructured means a new activity source can write whatever is most legible for its own kind of event without a schema change.

In the prototype these entries are synthetic, plus one appended per simulated tab visit. In a real build the producer is a `tabs.onUpdated` listener classifying visited hosts against a local topic table — never a remote lookup, since sending the user's browsing history to a classification service would defeat the entire premise.

### 10.2 The activity log is ephemeral, and that is load-bearing

A browsing history is more revealing than the ad profiles this tool exists to critique. A privacy tool that accumulates one — even locally, even with good intentions — has become the thing it is arguing against. So the storage layer is deliberately the weakest one available:

| Mechanism | Prototype | Real extension |
|---|---|---|
| Persistence | none — a plain in-memory array | `chrome.storage.session`, never `local`, never `sync` |
| Expiry | rows past the retention window are dropped on every inference | same |
| Deletion | per entry, per topic, or all; plus a reset of captured platform profiles | same |
| Derived data | recomputed on every render, never cached | same |

Four properties follow, each of which is a constraint on future changes rather than an implementation detail:

**No persistence, and no recovery path.** Closing the tab destroys the log. A recovery path would be a stored copy, so there deliberately isn't one. In the extension, `chrome.storage.session` gives exactly the right semantics — it survives the service-worker restarts described in §2 but not browser close. MV3's non-persistent worker, normally an obstacle, is aligned with this design.

**Expiry deletes rather than summarises.** This is the non-obvious one. A retention rollup — "was interested in cars last month" — would be smaller than the rows behind it but no less sensitive, and it would quietly outlive the deletion the user asked for. There is therefore no rollup anywhere in the pipeline. A consequence worth preserving: narrowing the retention window destroys rows irreversibly, and widening it back does not restore them. Destructive and irreversible is correct for a delete control.

**Deletion is partitioned by topic, not all-or-nothing.** There is no single "your history" blob. Forgetting one interest doesn't cost the others, so the user is never forced to choose between granularity and privacy.

**Nothing derived is cached.** Scores, confidences, summary sentences and the export are all recomputed from `ACTIVITY` on each render. This is what makes deletion actually mean deletion — a cached score would leave a conclusion standing after its evidence was destroyed, which is precisely the failure mode this product exists to criticise. It is the same invariant as §10.5, viewed from the other side.

The tradeoff is real and should not be papered over: a longer window would produce better inferences and catch slow-building interests that 14 days misses. The judgement is that a tool making its point from two weeks of data makes the argument more effectively than one that needs a year of history to do it, and the user-selectable window lets anyone who disagrees choose otherwise while the default protects whoever never opens settings.

### 10.3 Export as the only durable copy

Because nothing persists, an export is the sole route to durable data — which makes it a deliberate user action producing an ordinary file they own and can delete like any other. The asymmetry is the design: persisting takes an explicit act, forgetting is the default. That is the inverse of every platform whose profiles this tool displays, and it is the clearest available demonstration of the alternative.

### 10.4 Scoring

```
score = (distinctDays × 2 + totalVisits) × recency

recency = daysSinceLast ≤ 7  → 1.0
          daysSinceLast ≤ 21 → 0.6
          otherwise          → 0.3
```

Distinct days are weighted double because they discriminate better than raw volume: eight page views in one evening is a single burst of curiosity, while three visits across three days is sustained intent. Raw visit count still contributes, but as the weaker term.

Recency decays in steps rather than continuously, and floors at 0.3 instead of zero. A step function is chosen over exponential decay purely so the user can reproduce the number by hand — a score nobody can recompute is as opaque as the platform profile this is meant to critique. The floor exists because stale interest is weak evidence, not absence of evidence.

Confidence is a separate categorical rule (`high`: 3+ distinct days and recent; `medium`: 2+ days; `low`: otherwise) rather than a threshold on `score`, so that a high score driven by one frenzied afternoon still reports as low confidence. Volume and reliability are different claims and the UI shows them separately.

### 10.5 The evidence invariant

`inferProfile()` returns the contributing `ActivityEntry` rows on every result object, and the renderer builds both the summary sentence and the expandable list from that same array. There is no path that emits a conclusion without its sources, because the conclusion and the sources are the same object.

This is the one property worth protecting during any future change to this file. If scoring later moves to something less legible — embeddings, a model, cross-user signals — the requirement isn't that it stay simple, it's that it stay able to name the rows that produced each output. An inference that can't do that shouldn't be displayed, because the product's only claim to being different from an ad platform is that its guesses are checkable.

### 10.6 Agreement with disclosed data

`disclosedMatch()` does a loose token comparison between an inferred category and the stored platform tags, and the UI badges the overlaps. The loose match is intentional: it's a display affordance pointing the user at a comparison to make themselves, not a claim of equivalence, so a false positive costs little while a missed overlap hides the interesting case. The genuinely informative result is the *absence* of a match — a platform interest with no support anywhere in the user's own activity.

---

## 11. Export

A single synchronous `buildExport()` assembles disclosed profiles, the full activity log, the inferred profile with evidence, and current session counters into one object. Three serializers render it: JSON (complete, canonical), HTML (the same data formatted for reading), CSV (activity log only, for spreadsheets). They share one builder so the formats cannot drift apart.

Delivery is `Blob` → `URL.createObjectURL` → synthetic anchor click → `revokeObjectURL`. No server is involved at any point, which is a design constraint rather than an implementation detail: an export feature whose purpose is demonstrating local-only data handling must not acquire a network dependency. The HTML serializer escapes all interpolated values via `esc()` — the activity log contains hostnames and page titles that originate from pages the user visited, so it is untrusted input being written into a document the user will open.

The whole file is regenerated on each click rather than cached, so an export always reflects the state at the moment it was requested.
