---
name: Track new Osmaura editions and diff them
description: >-
  Watch the Osmaura Prospect API for newly published editions and revisions,
  diff them against what you already hold, and surface only genuinely new or
  changed prospects — using edition history and ETags rather than re-reading
  everything.
api: openapi/osmaura-prospect-openapi.yml
operations:
  - listProspectEditions
  - getProspects
generated: '2026-08-14'
method: generated
source: >-
  openapi/osmaura-prospect-openapi.yml +
  https://dashboard.osmaura.com/signals/docs
---

# Track new Osmaura editions and diff them

Osmaura does not push. There are no webhooks, no streaming endpoint, and no
AsyncAPI — the only way to learn that something new exists is to ask. This skill
is the polite way to ask.

## The two things that change

1. **A new `edition_date`** — a fresh batch of prospects.
2. **A new `revision` on an existing `edition_date`** — the provider corrected or
   republished an edition you may already have processed. Revisions start at 1 and
   increase.

Both matter. Tracking only `edition_date` will silently miss corrections.

## Step 1 — Read the history, not the payload

Start with `listProspectEditions`, which is cheap and carries no dossiers:

```
GET https://dashboard.osmaura.com/v2/prospect-editions?limit=30
Authorization: Bearer $SIGNALS_API_KEY
```

Each entry has `edition_date`, `published_at`, `revision`, and `count`. Compare
the `(edition_date, revision)` pairs against your local state.

- Pair you have not seen → fetch it.
- Pair you have seen → skip it entirely.
- Same `edition_date`, **higher** `revision` → re-fetch and re-diff; treat your
  stored copy as stale.

`limit` accepts 1–90 and defaults to 30. Keep it small on a frequent schedule.

## Step 2 — Fetch only what is new

For the latest edition, use the conditional form and let the server tell you
whether anything changed:

```
GET https://dashboard.osmaura.com/v2/prospects
Authorization: Bearer $SIGNALS_API_KEY
If-None-Match: "<last stored etag>"
```

`304` → nothing to do. `200` → a new edition or revision; store the body and the
new `ETag`.

For a specific historical or revised edition, name the date:

```
GET https://dashboard.osmaura.com/v2/prospects?date=2026-07-21
Authorization: Bearer $SIGNALS_API_KEY
```

Only dates that appeared in step 1 will resolve. Anything else returns `404`.

## Step 3 — Diff on `id`, not on position

Every dossier carries a stable `id` (e.g. `sig_…`). Rank is **not** an identity —
`analysis.ranking.rank` is a position within one edition and will move between
editions for the same prospect.

Diff like this:

- `id` present now, absent before → **new prospect**.
- `id` present in both → compare `analysis.ranking.disposition`,
  `analysis.ranking.rank`, `analysis.ranking.confidence`, and
  `analysis.why_now.trigger_date`. A disposition moving toward `qualified`, or a
  newer `trigger_date`, is the interesting change.
- `id` absent now, present before → the prospect dropped out of this edition. That
  is not a statement that it was wrong; check `coverage` before drawing conclusions.

## Step 4 — Report changes honestly

When you summarize a diff for a human:

- Quote `analysis.summary` and `analysis.why_now.narrative` **as Osmaura's
  assessment**, not as established fact.
- Carry the evidence with the claim: `analysis.why_now.evidence_ids` point at
  record ids under `data`, and each of those records has a `source` block with an
  `official_page_url` and a `record_locator`. Include the link.
- Surface `analysis.counterevidence` and `analysis.limitations` alongside the
  positive case. They exist in the schema because the provider intends them to be
  read.
- Include `coverage.data_through` so the reader knows how fresh the underlying
  government data is — it commonly lags the edition date by a quarter.

## Scheduling

There is no published rate limit, no `429`, and no `Retry-After` header, so there
is no runtime signal telling you to slow down. Pick a cadence from the product
shape instead: editions are periodic and typically 20 prospects, so polling
`listProspectEditions` a small number of times per day is ample. Always send
`If-None-Match` on `getProspects`.

## Errors worth special handling in a scheduled job

- `402` — the organization plan lapsed. **Stop the schedule and alert a human.**
  Retrying cannot fix a billing state.
- `401` — someone rotated the key; rotation revokes the previous key immediately.
  Stop and request the new secret.
- `503` — account verification is temporarily unavailable. Retry with backoff;
  this one is transient.
- `404` on a dated fetch — you asked for a date that is not published. Re-read the
  history rather than retrying the same date.
