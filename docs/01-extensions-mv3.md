# 1. Browser extensions & Manifest V3

## Level 1 — the high level

A browser extension is a small web app with elevated privileges. It's HTML, CSS and JS,
plus a `manifest.json` that declares what it is and what it's allowed to touch. There's no
build step required and no compilation — you can load an unpacked folder straight into
Chrome from `chrome://extensions` with Developer Mode on.

An extension is made of pieces that run in **different places with different powers**, and
understanding that split is most of the learning curve:

| Piece | Runs where | Can it see the page? | Can it use extension APIs? |
|---|---|---|---|
| **Service worker** (background) | Its own worker context | No | Yes, nearly all |
| **Content script** | Injected into a web page | Yes — the DOM | Only a small subset |
| **Side panel / popup** (UI) | Its own document | No | Yes, nearly all |

Nothing shares memory with anything else. They communicate by passing messages. Your
`architecture.md` already has this shape right: content script extracts → sends message →
service worker stores → panel reads storage and renders.

## Level 2 — what MV3 changed, and why it will bite you

Manifest V3 replaced MV2. The change that matters most to you:

**The background page is now a non-persistent service worker.** In MV2, your background
script was a page that stayed alive for the browser session, so you could keep state in a
variable. In MV3 the worker is **killed after roughly 30 seconds of inactivity** and
restarted from scratch when a new event arrives. Restarted means re-executed from the top,
with every variable gone.

The practical rule that follows:

> **Never keep state in a module-level variable in the service worker.** It will
> disappear, and it will disappear *intermittently* — working fine while you're actively
> clicking around during development, then failing for real users who leave the browser
> idle.

All state lives in `chrome.storage`. Your event listeners must also be **registered
synchronously at the top level** of the worker script, not inside a callback or after an
`await` — if the worker restarts to handle an event and your listener isn't registered by
the time the script finishes its first tick, the event is lost.

```js
// ✅ correct — top level, synchronous registration
chrome.runtime.onMessage.addListener((msg) => { handle(msg); });

// ❌ broken — listener registered too late after a restart
(async () => {
  const config = await chrome.storage.local.get('config');
  chrome.runtime.onMessage.addListener(...);   // may never see the event
})();
```

The second MV3 change: **no remotely-hosted code**. Everything executable ships in the
package. No CDN script tags, no `eval`. Not a constraint you'll feel here — you have no
dependencies — but it's why extension tutorials that pull in a library from unpkg don't
work.

Third: **`chrome.webRequest` lost its blocking ability.** This one gets its own document —
see [doc 2](02-observing-behaviour.md).

## Level 3 — the depth you need for PrivaTrace

### Permissions, and why yours are unusually clean

Permissions come in two kinds, and the distinction is worth internalising because it's
what your install prompt says to users:

- **`permissions`** — API access. `storage`, `tabs`, `sidePanel`. Mostly invisible to users.
- **`host_permissions`** — which *sites* your code can touch. This is what generates the
  scary "Read and change your data on all websites" warning.

Your `architecture.md` scopes `host_permissions` to three exact origins instead of
`<all_urls>`. That's the right call and it's worth knowing exactly what you buy with it:
the content script becomes **physically incapable** of running elsewhere. It's not a policy
your code enforces and could get wrong — the browser won't inject it. When you later add
the observed panel, you'll face pressure to widen this, because watching trackers on
arbitrary pages genuinely does need broad access. Consider `optional_host_permissions` and
requesting at runtime, so the disclosed-panel feature works on a narrow grant and the user
opts into the broader one separately.

### Storage

`chrome.storage.local` — per-device, survives restarts, ~10MB by default (more with the
`unlimitedStorage` permission). `chrome.storage.session` — **in-memory, cleared when the
browser closes**, and not accessible to content scripts by default. Given the direction
this project has taken on data handling, `storage.session` is probably where your activity
log belongs, if it's stored at all. See [doc 7](07-ephemeral-data-design.md).

`chrome.storage.sync` exists and you should not use it. It leaves the device and goes
through Google's servers. For a privacy tool holding a browsing history that's
disqualifying, and it's also tiny (~100KB) and quota-limited.

All storage APIs are async and promise-based in MV3.

### Debugging, concretely

This is where you'll lose the most time if nobody tells you:

- **The service worker has its own console.** `chrome://extensions` → your extension →
  "Inspect views: service worker". Logs there do *not* appear in the page console. A
  worker showing as "inactive" is normal, not broken.
- **Content script logs go to the page's console**, mixed in with the page's own noise.
  Prefix yours.
- **Reloading matters and is inconsistent.** Changing the service worker or manifest needs
  an extension reload (the ↻ icon). Changing a content script needs an extension reload
  *and* a reload of any page it's injected into. Stale-code confusion is the single most
  common early time-sink.
- **Errors in the worker are easy to miss entirely** — they surface as a small "Errors"
  badge on the extensions page rather than anywhere you're looking.

### Where to stop

You don't need: declarative content, offscreen documents, the side-panel API beyond
opening it, chrome.alarms (you have no scheduled work by design), or the Web Store
publishing flow until you decide to distribute.

## Sources worth reading firsthand

- Chrome's **"Migrate to Manifest V3"** guide — read this even though you're starting
  fresh; it's the clearest statement of what changed and why, and it inoculates you
  against outdated tutorials.
- The **`chrome.storage`** and **`chrome.runtime`** API reference pages.
- Chrome's extension samples repo — small, current, MV3-correct working examples.
