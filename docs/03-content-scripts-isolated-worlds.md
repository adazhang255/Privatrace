# 3. Content scripts & isolated worlds

The fiddliest part of the project. Budget real time here, and read this before you write
the fingerprint detector, because the naive approach silently does nothing.

## Level 1 — the high level

A **content script** is your JS, injected by the browser into a web page. It can read and
modify that page's DOM. It's how your extractors read interest tags off Google's ad
settings page.

The thing that trips everyone up:

> A content script shares the page's **DOM** but not the page's **JavaScript**.

Two separate JS environments, same document. Yours is the **isolated world**; the page's is
the **main world**. They each have their own `window`, their own globals, their own copies
of the built-in objects. If the page defines `window.dataLayer`, your content script sees
`undefined`. If you define a variable, the page can't see it. Both see the same `<div>`s.

This is a security boundary, and a good one — it means a hostile page can't reach into your
extension, and your code can't be broken by the page redefining a global.

## Level 2 — why this decides your fingerprint feature

Fingerprint detection means noticing that the page called `canvas.toDataURL()`. The obvious
approach is to wrap the function:

```js
// In a content script. Looks right. Detects NOTHING.
const orig = HTMLCanvasElement.prototype.toDataURL;
HTMLCanvasElement.prototype.toDataURL = function(...args){
  report('canvas.toDataURL() called');
  return orig.apply(this, args);
};
```

This runs, throws no error, and never fires. You patched **your** isolated world's
`HTMLCanvasElement`. The page has its own, untouched. You've instrumented a copy nobody
calls.

To see the page's calls you must run code **in the main world**. Two routes:

### Route A — declare it in the manifest (MV3, preferred)

```jsonc
"content_scripts": [{
  "matches": ["https://example.com/*"],
  "js": ["inject/fingerprint-hooks.js"],
  "world": "MAIN",
  "run_at": "document_start"
}]
```

`"world": "MAIN"` is the modern, supported way. Verify support in your target Chrome
version — it's relatively recent, and it's the kind of thing worth confirming in the docs
rather than trusting a document.

### Route B — `chrome.scripting.executeScript` with `world: 'MAIN'`

Programmatic equivalent, for when you want to inject conditionally. Needs the `scripting`
permission.

### The old trick, and why to avoid it

Pre-MV3 code injects a `<script>` tag into the page. It still works, but MV3's
no-remote-code rules and page CSP make it fragile, and the supported `world: "MAIN"` route
exists now. Don't learn the old pattern.

## Level 3 — the depth you need

### Main-world code has no `chrome.*` APIs

This is the catch. Main-world code can see the page's calls but **cannot call
`chrome.runtime.sendMessage`** — it's not an extension context anymore, it's just page JS.
So reporting a detection takes a relay:

```
main-world hook  →  window.postMessage  →  isolated content script  →  chrome.runtime.sendMessage  →  service worker
```

Two consequences:

**Validate at the boundary.** `window.postMessage` is a public channel — the page can post
messages too, pretending to be your hook. Check `event.source === window`, check `event.origin`,
and namespace your message type. Never pass anything from a postMessage straight into a
privileged API without validating its shape. Treat it as untrusted input, because it is.

**Your hook is visible to the page.** Main-world code lives in the page's world, so the
page can detect it — `toDataURL.toString()` no longer returns `[native code]`. A page
determined to hide fingerprinting from you can. Accept it; don't claim completeness in the
UI.

### Timing: `run_at`

| Value | When | Use for |
|---|---|---|
| `document_start` | Before the DOM is built | **Required** for hooks — you must patch before page code runs |
| `document_end` | DOM parsed, subresources maybe not | Rarely what you want |
| `document_idle` | After load settles | Your *extractors* (per `architecture.md` §3) |

The asymmetry matters: hooks must be as early as possible or you miss calls; extractors
want to be late or the data isn't rendered yet. Different scripts, different `run_at`.

### SPAs and why extraction needs a MutationObserver

Your `architecture.md` §3 already gets this right, and it's the correct instinct for
exactly the right reason: those settings pages are client-rendered, so at `document_idle`
the DOM is an empty shell and the interest list arrives later via fetch. Parsing
immediately returns nothing — and "nothing" must never be written over good stored data.

Two refinements to what's in the architecture doc:

- **`observer.observe(document.body, {childList:true, subtree:true})` on a large SPA fires
  a lot.** Debounce the extraction attempt, or you'll run the selector query hundreds of
  times per second during page construction.
- **The 15-second timeout needs a distinct outcome from "extracted nothing".** "Timed out,
  never saw the list" and "found the list, it was empty" are different facts — the first is
  probably a broken selector, the second might be a genuinely empty ad profile. Your
  `status: 'empty' | 'error'` split is the start of this; make sure timeout is a third case
  and that the UI can tell the user which happened.

### What to instrument, if you do this

The realistic fingerprinting surface: `HTMLCanvasElement.prototype.toDataURL` and
`getImageData`, `AudioContext`/`OfflineAudioContext`, `WebGLRenderingContext.getParameter`
(vendor/renderer strings), `navigator.plugins` / `mimeTypes`, `document.fonts.check`, and
`navigator.hardwareConcurrency` / `deviceMemory`.

**Important caveat for honest UI:** these APIs all have legitimate uses. `toDataURL` is how
you implement "download this chart as PNG." A single call is not evidence of
fingerprinting; a burst of calls across several unrelated APIs with no visible feature to
justify them is suggestive. Your demo labels each call "flagged," which slightly
overclaims — consider flagging *patterns* rather than individual calls, or softening the
label to "fingerprintable API used."

### Testing it

Don't test against your own synthetic calls — they'll pass trivially and prove nothing.
Test against a real fingerprinting library (FingerprintJS is open source) or a live
demo page like amiunique.org, and check you catch what it actually does.

## Where to stop

You don't need: `chrome.scripting` beyond one-shot injection, CSS injection, shadow DOM
piercing, or cross-frame messaging — though note iframes are a real gap. Trackers often
live in iframes, and `all_frames: true` plus per-frame attribution is a whole extra layer.
Reasonable to declare out of scope, but say so rather than leaving the UI implying it's
covered.
