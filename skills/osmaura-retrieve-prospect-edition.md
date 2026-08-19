---
name: Retrieve the latest Osmaura prospect edition
description: >-
  Fetch the current published prospect edition from the Osmaura Prospect API,
  poll it efficiently with conditional requests, and read the dossiers without
  confusing observed records for analyst conclusions.
api: openapi/osmaura-prospect-openapi.yml
operations:
  - getProspects
  - listProspectEditions
generated: '2026-08-14'
method: generated
source: >-
  openapi/osmaura-prospect-openapi.yml +
  https://dashboard.osmaura.com/signals/docs
---

# Retrieve the latest Osmaura prospect edition

Osmaura publishes **editions** — dated, human-reviewed batches of ranked prospect
dossiers scoped to your organization. This is a read-only API: there is nothing to
create, update, or delete.

## Before you start

- You need a production read key from the Osmaura dashboard. It looks like
  `signals_live_…` and is shown **once**, at setup or after a rotation.
- The key is scoped to one account. **Never send an organization name or account
  id** — the API infers the account from the key, and no operation accepts one.
- The organization must have an active plan. Without one every call returns `402`.
- If you are an agent, ask for your **own** key rather than reusing a human's:
  the dashboard mints additional keys without revoking existing ones.

## Step 1 — Get the latest edition

Call `getProspects` with no parameters.

```
GET https://dashboard.osmaura.com/v2/prospects
Authorization: Bearer $SIGNALS_API_KEY
```

The response is a `prospect_edition` object with `edition_date`, `revision`,
`published_at`, `count`, and a `prospects` array in ranked order. A normal edition
contains 20 prospects.

**Store the `ETag` from the response headers.** You need it for step 3.

## Step 2 — Read a dossier correctly

Each item in `prospects` has exactly five top-level fields. The split between them
is a contract Osmaura publishes explicitly, and getting it wrong is the main way to
misuse this API:

| Field | What it is | How to treat it |
|---|---|---|
| `id` | Stable prospect id (e.g. `sig_…`) | Key for dedupe across editions |
| `prospect` | Normalized identity only | Fact |
| `data` | Observed records + deterministic statistics, by source | **Fact** — every government record carries a `source` |
| `analysis` | Rank, scores, summary, why-now, counsel assessment | **Conclusion** — attribute it, never restate as fact |
| `coverage` | What was checked, freshness, gaps | Read before trusting an absence |

Rank lives at `analysis.ranking.rank`, not at the top level.
`analysis.ranking.disposition` is one of `qualified`, `nurture`, `monitor`,
`suppress`, with `confidence` between 0 and 1.

**Verify before you repeat a fact.** A source code such as `dol_lca` means nothing
on its own. Use `source.official_page_url` plus `source.record_locator` (and, for
bulk data, `source.dataset.download_url` and `source.dataset.data_through`) to
point a human at the underlying record.

**Do not turn `counsel_analysis` into a boolean.** It reports `filings_reviewed`,
`named_counsel_filings`, and a standing limitation that public records cannot
reveal every private advisory relationship. "Observed pro se" is a scoped
observation about reviewed filings, not a claim that a company has no lawyer.

**Check `coverage` before concluding "nothing found."** It distinguishes zero from
unknown from not-checked, and lists `identity_warnings` and `known_gaps`.

## Step 3 — Poll without re-downloading

Editions are published on a cadence, not streamed. Send the stored ETag back:

```
GET https://dashboard.osmaura.com/v2/prospects
Authorization: Bearer $SIGNALS_API_KEY
If-None-Match: "<etag from step 1>"
```

A `304` means the edition has not changed — stop, and keep what you already have.
A `200` means a new edition or a new `revision` of the same `edition_date` was
published; replace your copy and store the new ETag.

There is no published rate limit and no `Retry-After` contract, so conditional
requests are how you stay a good citizen here.

## Step 4 — Reach a specific past edition

Do not guess dates. List what exists first with `listProspectEditions`:

```
GET https://dashboard.osmaura.com/v2/prospect-editions?limit=30
Authorization: Bearer $SIGNALS_API_KEY
```

`limit` accepts 1–90 and defaults to 30. The response gives `edition_date`,
`published_at`, `revision`, and `count` per edition. Then fetch one:

```
GET https://dashboard.osmaura.com/v2/prospects?date=2026-07-21
Authorization: Bearer $SIGNALS_API_KEY
```

Drafts and internal review state are never exposed — if it is not in the history
list, it does not exist for you.

## Errors

| Status | Meaning | Do this |
|---|---|---|
| `400` | Bad `date` or `limit` | Use `YYYY-MM-DD`; keep `limit` in 1–90 |
| `401` | Key missing, invalid, or revoked | A rotation revokes the old key immediately — get the new one |
| `402` | No active organization plan | Billing state, not a bad request. Stop retrying; tell a human |
| `404` | No published edition for that date | Call `listProspectEditions` and pick a real date |
| `503` | Account verification temporarily unavailable | Transient — retry with backoff |

Every error body is `{"error": {"code": "...", "message": "..."}}`. This is **not**
RFC 9457 `application/problem+json`; there is no `type`, `title`, or `detail`
field. See `errors/osmaura-problem-types.yml`.

## Do not

- Do not retry a `402` — it will not clear without a billing change.
- Do not use `/v1/signals` or `/v1/signal-editions` for a new integration. Both are
  marked deprecated in the spec and return the compact legacy shape.
- Do not present `analysis` narratives as verified facts about a company.
