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
