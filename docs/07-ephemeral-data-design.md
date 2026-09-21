# 7. Designing for ephemeral data

This is the doc that describes a decision you've already made and that is now implemented
in `demo.html`. It's here because it's the project's actual differentiator, and because the
temptation to walk it back will be constant.

## Level 1 — the problem in one sentence

To tell you why you're seeing car ads, PrivaTrace needs your browsing history — which
means a privacy tool would be assembling the single most revealing dataset a person owns,
in order to complain that other people assemble it.

That's not a minor irony. It's the kind of thing that, if you shipped it and someone
noticed, would be the only thing anyone remembered about the project.

## Level 2 — the resolution: don't hold it

The decision, as now implemented:

> **Nothing persists. Everything expires. Everything is deletable, granularly, at any time.
> The only durable copy is one the user explicitly asks for.**

Four mechanisms, in `demo.html`:

### 1. No persistence at all

The activity log is a plain in-memory JS array. No `localStorage`, no `IndexedDB`, no
`chrome.storage`. Close the tab and it's gone, with no recovery path — deliberately no
recovery path, because a recovery path is a stored copy.

In the real extension this is `chrome.storage.session` (in-memory, cleared when the browser
closes) rather than `chrome.storage.local`. Never `chrome.storage.sync`, which leaves the
device entirely.

### 2. Automatic expiry

A retention window (default 14 days, user-selectable 3/7/14/30) drops rows past the cutoff.
`inferProfile()` prunes before computing, so an expired row can never reach a conclusion.

**Expiry deletes rows rather than summarising them first.** This is the subtle part and the
easy thing to get wrong. A rollup — "was interested in cars last month" — would be smaller
than the rows behind it but no less sensitive, and it would quietly outlive the deletion the
user asked for. So there is no rollup. Nothing is aggregated for retention purposes.

Note a consequence, which is intended: narrowing the window from 30 days to 3 destroys
data, and widening it back doesn't restore it. Destructive and irreversible is the correct
behaviour for a delete control.

### 3. Granular, partitioned deletion

Three levels: individual entry (in the evidence list), whole topic ("Forget this" per
category), and everything. Plus `deleteEverything()`, which also drops captured platform
profiles and resets the session counters.

The partitioning is a real design point, not just convenience. There is no single "your
history" blob that can only be deleted all-or-nothing — so forgetting one interest doesn't
cost you the rest, and the user isn't forced to choose between granularity and privacy.

### 4. Derived data is never cached

Everything shown — scores, confidence levels, the summary sentences, the export — is
recomputed from `ACTIVITY` on every render. Nothing derived is stored anywhere.

This is what makes deletion *actually* mean deletion. If a score were cached, deleting its
evidence would leave the conclusion standing, and the tool would be asserting something
about the user with the receipts destroyed — the exact failure mode it exists to criticise.

## Level 3 — the depth you need

### Why this is a feature, not a limitation

The instinct is that ephemerality is a cost you pay for principle. It isn't, for this
product:

- **It makes the export meaningful.** Because nothing persists, exporting is the only way
  to make data durable — so it's a deliberate act producing a file the user owns and can
  delete like any other. Persisting takes effort; forgetting is the default. That's the
  right way round, and it's the inverse of every platform you're critiquing.
- **It makes the inference honest.** A 14-day window with visible expiry means the tool's
  profile is *obviously* partial. Which is true, and which is exactly what you want the
  user thinking about: if 12 rows over two weeks produces "high confidence: car shopping,"
  what does a platform with years of data across every device produce?
- **It removes your entire liability surface.** No stored history means no breach, no
  subpoena, no support burden around deletion requests, no GDPR data-subject-access
  machinery. You can't lose what you don't have.

### The honest tradeoff

State it plainly rather than pretending it away: **longer history makes better inferences.**
A 90-day window would spot seasonal and slow-building interests that 14 days misses, and
the inferences would be more like what platforms actually do — which is arguably more
educational.

The judgement here is that a tool which demonstrates the point with two weeks of data makes
the argument *better* than one that needs a year of your history to do it. And the
user-selectable window means someone who disagrees can choose 30 days, while the default
protects the person who never opens settings.

### Pressure you'll feel to break this

Expect all of these, and the answer is no:

| "We should…" | Why it breaks the design |
|---|---|
| …persist so the profile survives a restart | That's the stored history, back again |
| …keep a small summary when rows expire | The summary is the sensitive part |
| …sync across devices | Data leaves the device |
| …let users log in to save their profile | Now you have a server and a database of histories |
| …cache scores for performance | Deletion stops meaning deletion |
| …collect anonymous analytics | A privacy tool with telemetry is the joke that writes itself |

If you ever do add persistence, the requirement is that it be **opt-in, local-only, and
subject to the same expiry and granular deletion** — not that the defaults change.

### If you build this for real

- Use `chrome.storage.session`, not `local`, for the activity log.
- The **service worker restarting wipes in-memory state anyway** (see
  [doc 1](01-extensions-mv3.md)) — MV3's non-persistence, normally an obstacle, is aligned
  with you here. `storage.session` survives worker restarts but not browser close, which is
  exactly the semantics you want.
- Put the retention control and the delete buttons somewhere prominent, not buried in
  settings. In `demo.html` "What's being held right now" is the *first* panel on the data
  page, above the inference. That ordering is intentional: the user learns what's held
  before they see what was concluded from it.
- Say what you do, in the product, in plain language. The green panel states it in two
  sentences. Claims like this are cheap to make and easy to check — which means making them
  visibly and specifically is a commitment, and that's the point.
