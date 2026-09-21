# 4. Extracting data from pages you don't control

This is your "Disclosed by platforms" panel, and it's the project's structural weak point.
Not because the code is hard — each extractor is ~30 lines — but because it depends on
three companies' undocumented, frequently-redesigned HTML.

This document is a *practice*, not a body of facts. There's nothing to memorise; there's a
way of working that makes the fragility survivable.

## Level 1 — the high level

The interest tags are on the page. You query the DOM for them. Done.

The problem is that `class="mR2gOd QJZWLd"` is machine-generated, changes on every deploy,
and means nothing. The selectors in `architecture.md` §4 are explicitly labelled
placeholders, which is honest — but it also means **none of the real work is done yet**,
and you can't know how hard it is until you open DevTools on each page.

## Level 2 — writing selectors that survive

The ranking, most to least durable:

1. **ARIA and semantic attributes** — `[role="listitem"]`, `[aria-label]`. These encode
   *meaning*, and meaning survives a visual redesign. They're also legally load-bearing
   for accessibility at companies this size, so they're maintained.
2. **Stable-looking data attributes** — `[data-testid]`, `[data-*]`. Often survive because
   the platform's own tests depend on them. Sometimes stripped in production builds.
3. **Document structure** — "the list inside the section whose heading contains
   'interests'". Verbose but semantic.
4. **Text content** — find the heading by its text, then walk relative to it. Breaks on
   localisation; anchor on the user's own locale only.
5. **Generated class names** — `.mR2gOd`. Will break. Use only as a last resort and expect
   to re-derive it.

A composite approach beats any single selector: find a semantic anchor, then search within
it. That way a restyle inside the container doesn't break you.

```js
// More durable than a single deep selector chain
function extract(doc){
  const section = [...doc.querySelectorAll('section, [role="region"]')]
    .find(el => /interest|topic|categor/i.test(el.querySelector('h1,h2,h3')?.textContent || ''));
  if (!section) return [];
  return [...section.querySelectorAll('[role="listitem"], li')]
    .map(el => el.textContent.trim())
    .filter(Boolean);
}
```

## Level 3 — the depth you need

### Distinguish the three failure modes, always

This is the most important idea in this document. Your architecture doc has `ok` / `empty`
/ `error`; the real taxonomy is four:

| Outcome | Means | Correct response |
|---|---|---|
| **Found data** | Selector matched, items present | Store it |
| **Found the container, it's empty** | Genuinely no interests on the account | Store the empty result — this is real data! |
| **Never found the container** | Selector is broken, or page didn't load | **Keep old data.** Flag "sync failed" |
| **Threw** | Bug in your extractor | Keep old data. Flag, and log the error |

Conflating the middle two is the bug that silently wipes a user's panel after a redesign,
and it's the single highest-value invariant in the extension. `architecture.md` §3 says
"never overwrite good data with an empty result" — refine that to "never overwrite with a
result you can't distinguish from failure," and make "genuinely empty" a positively
identified state rather than a fallthrough.

### Golden-file tests

The practice that makes this maintainable:

1. Open each settings page in your own browser, logged in.
2. **Save the rendered HTML** (DevTools → Elements → right-click `<html>` → Copy outerHTML
   — you want post-JS DOM, not `view-source`).
3. **Scrub it.** These files contain your real interests, possibly your name, email, and
   session tokens. Redact before it goes anywhere near a repo. Substitute obviously fake
   interest labels.
4. Commit the scrubbed fixture. Write a test: load fixture → run extractor → assert the
   expected labels come out.

Now a redesign breaks a test you can see, instead of a user's panel you can't. Re-capture
fixtures periodically; when a platform redesigns, the fixture update and the selector fix
are one commit.

This also lets you develop the extractor offline against a file, which is dramatically
faster than reloading a live settings page repeatedly — and avoids hammering it, which is
the sort of traffic pattern you'd rather not generate.

### The strategic point: prefer the export path

Everything above manages fragility. This *removes* it.

Google Takeout and Meta's "Download Your Information" both hand you your ad-interest data
as a structured file, through a documented, supported, explicitly-sanctioned channel built
for exactly this purpose. Parsing a ZIP you downloaded is stable in a way scraping can
never be — no selectors, no redesigns, no terms-of-service question at all.

`architecture.md` §8 defers this ("schema supports it, nothing produces it") and §5 already
reserves `sourceType: 'scrape' | 'export'`. I'd invert the priority:

> **Make import the primary path and scraping the convenience layer.**

The tradeoff is real and worth stating plainly: scraping gives one-click freshness, import
takes the user through a manual download that can take hours to be ready. So scraping is
the better *experience*. But import is the better *foundation* — it can't break, and it's
unambiguously permitted. Build import first, so that when a scraper breaks the product
degrades to "slower" rather than "broken."

### Terms of service, briefly

Reading a page you're logged into and looking at, with no scheduling and no headless
requests, is the most defensible end of automated access. But most platforms' ToS use broad
"automated means" language that doesn't carve out your case. For personal use this is a
non-issue in practice. Before distributing, get a real opinion — and note that the import
path sidesteps the question entirely, which is one more argument for it.

## Where to stop

You don't need: headless browsers, a scraping framework, proxies, or any
anti-bot-evasion technique. If you find yourself wanting those, you've left the design
this project committed to — see `architecture.md` §9.
