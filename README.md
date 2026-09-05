# Source Watch

An n8n monitoring workflow built around one question most pipelines never ask:

> **How would you know if this stopped working?**

Not crashed. Stopped. Still green, still running every 30 minutes, still
returning HTTP 200 — and returning nothing, for nine days, while everyone
assumed it was quiet because there was nothing to report.

That failure has no error message. It is the one this workflow is built to catch.

```
workflow/source-watch.json    ← import this into n8n

Verified on n8n 2.35.7 / Node 22.23.2 — imported, executed, dedupe and
heartbeats confirmed from the stored data. Details at the bottom.
```

---

## What it does

```
Every 30 min → Sources → Fetch (retry, never-error) → Check → Build message → Telegram
```

Four checks, in order:

| # | Check | Outcome |
|---|---|---|
| 1 | Did the request succeed? | HTTP 4xx/5xx → **loud** |
| 2 | Did the body parse into items? | unparseable → **loud** |
| 3 | Is anything new? | nothing new → **quiet, and that's fine** |
| 4 | **Has a source been quiet longer than its own limit?** | → **loud** |

Check 4 is the one that matters. Checks 1–3 catch a source that breaks.
Check 4 catches a source that keeps answering politely with an empty hand.

Each source declares its own tolerance, because "quiet" means different things
in different places:

```js
{ id: 'hn-jobs', silentAfterHours: 6 }        // busy source, 6h of nothing is wrong
{ id: 'example-feed', silentAfterHours: 24 }  // slow source, a day is normal
```

A single global threshold would either spam you about the slow source or stay
silent about the broken fast one.

---

## The bug this workflow shipped with, and why it is in the README

The first draft read the source metadata straight off the HTTP node:

```js
const source = entry.json.source || entry.json;   // wrong
const id = source.id || 'unknown';
```

The Fetch node runs with `fullResponse: true`, so its output is
`{ body, headers, statusCode }`. The `id`, `label` and `silentAfterHours`
fields set upstream are **not in it**.

So `source.id` was `undefined`, every source became `'unknown'`, all of them
shared one heartbeat slot, and the silence check — the entire reason this
workflow exists — could never fire for any individual source.

**Nothing errored. The workflow ran green.** An empty run looked exactly like
a healthy run with no news.

The fix is to pull the metadata back by position:

```js
const sources = $('Sources').all().map((i) => i.json);
// ...
const source = sources[idx] || {};
```

I am documenting it because it is a perfect specimen of the class of bug this
workflow is about: **the failure that produces no error.** A monitoring
pipeline whose monitoring is broken is worse than no pipeline, because it
manufactures confidence.

---

## Design decisions

**One dead source must not stop the others.**
`neverError: true` plus `onError: continueRegularOutput` means a failed request
arrives as data. Without it, one 500 kills the run and the other sources are
never checked — so a single broken source blinds you to all of them.

**Retry, but only three times, with a gap.**
`retryOnFail`, `maxTries: 3`, `waitBetweenTries: 5000`. Retrying forever turns
a slow source into a stuck workflow. Retrying instantly turns a rate limit into
a ban.

**Deduplicate on a fingerprint, not on position.**
Sources reorder, repaginate and re-emit. The fingerprint hashes whatever
identity fields exist (`id`, `url`, `title`, `guid`) and falls back to the whole
object. Position-based dedupe re-sends the same item every time the feed shifts.

**Prune the store.**
Seen-fingerprints are dropped after 30 days. `$getWorkflowStaticData` is
persisted with the workflow; an unbounded dictionary there grows until
something slow and confusing happens months later.

**Alerts look different from items.**
`SILENT —` and `BROKEN —` are formatted differently from new items on purpose.
Someone who receives twenty notifications a day stops reading them. The two
that mean *this is broken* have to be distinguishable from the eighteen that
mean *here is a thing*.

**The run summary is computed and then dropped.**
It is useful in the execution log and useless as a message. Not everything you
measure should be sent to a human.

---

## Verified

Imported and executed on **n8n 2.35.7, Node 22.23.2**.

```
Run manually                     success    1 item
Sources                          success    2 items
Fetch source                     success    2 items
Check, dedupe, detect silence    success    1 item     15 ms
Build messages                   success    0 items
```

Then read straight out of the workflow's stored static data:

```
fingerprint store : 600 entries
source heartbeats : 2   hn-jobs, example-feed
second run        : 0 new items
```

Three things that proves:

- **The metadata fix works.** The heartbeats are keyed `hn-jobs` and
  `example-feed`. Before the fix there would have been a single `unknown`
  key, and the silence check could not have fired for any individual source.
- **Deduplication works.** 600 fingerprints stored on the first run; the
  second run produced zero items and sent nothing.
- **Build messages produced 0 items on a quiet run** — which is correct.
  Nothing new, nothing broken, no notification. Silence in the right place.

### Two more things that only showed up on a real machine

**1. A hand-written workflow will not import.**

```
SQLITE_CONSTRAINT: NOT NULL constraint failed: workflow_entity.id
```

An export from the n8n UI carries a top-level `id`. A file assembled by hand
does not, and `import:workflow` rejects it without saying which field is
missing. Adding `"id"` at the root fixed it.

**2. A workflow with only a Schedule Trigger cannot be run by hand.**

```
Missing node to start execution
Please make sure the workflow you're calling contains an Execute Workflow Trigger node
```

`n8n execute` needs a manually-triggerable start node. The Schedule Trigger
alone is not one.

This is more than a CLI detail. **A workflow nobody can run on demand is a
workflow nobody verifies.** It sits there being green on a schedule, and the
only way to find out whether it still works is to wait and hope. Both triggers
are now wired to the same entry point.

## What is still not verified

- Telegram is not exercised. It needs credentials and `TELEGRAM_CHAT_ID`; the
  run above reached the node with `onError: continueRegularOutput`.
- The silence alert has not fired in anger — it cannot, on a fresh store,
  until a source has been quiet past its own limit. The clock starts on run one.
- The two example sources are public test endpoints. Replace them.

---

## Setup

1. n8n → **Workflows → Import from File** → `workflow/source-watch.json`
2. Open **Notify**, add Telegram credentials
3. Set `TELEGRAM_CHAT_ID` in your environment
4. Open **Sources**, replace the two examples with real ones
5. Run manually once. The first run only starts the heartbeat clock — silence
   alerts can only appear from the second run onwards.

---

## Related

- [freelance-radar](https://github.com/SystSezer/freelance-radar) — the Python
  original: five sources, pluggable connectors, LLM scoring, 1,100+ items
  processed.
- [sirket-arastirma](https://github.com/SystSezer/sirket-arastirma) — a research
  pipeline whose README opens by explaining that the first version scored 10%
  and why the fix was a better source rather than better code.

---

Sezer Kıraş · [systsezer.github.io/portfolyo](https://systsezer.github.io/portfolyo)
