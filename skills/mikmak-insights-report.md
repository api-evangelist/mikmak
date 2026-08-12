---
name: mikmak-insights-report
description: >-
  Pull MikMak commerce intelligence — Purchase Intent, Attributable Sales, pricing intelligence,
  shoppable recipe performance — by discovering a report's fields and filters first, then
  running or exporting it.
api: MikMak Insights API
base: https://api.mikmak.ai
operations:
  - POST /reporting/authenticate
  - POST /reporting/v1/custom_report_fields
  - POST /reporting/v1/custom_report_filters
  - POST /reporting/v1/custom_report_advanced_filters
  - POST /reporting/v1/custom_report_single_filter
  - POST /reporting/v1/custom_report
  - POST /reporting/v1/custom_report/export
generated: '2026-08-12'
method: generated
source: openapi/mikmak-insights-api-openapi.yml, https://docs.mikmak.ai/reference/mikmak-insights-api
---

# Run a MikMak Insights report

Three report families share one interaction shape: **discover fields → discover filters → run →
(optionally) export**. Never hand-build a report body; ask the API what it accepts.

Every operation is a `POST`, including the read-only discovery calls. Credentials are tied to a
single account and every response returns only that account's data.

## Step 0 — authenticate

`POST /reporting/authenticate`

The only unauthenticated operation on this API. Returns `access_token`, `token_type`,
`expires_in` and `expires_at`. Cache it until shortly before `expires_at`.

This is a **separate** token surface from the Commerce API's
`https://api.mikmak.ai/commerce/v1/oauth/token`. A Commerce token will not work here.

Subsequent calls carry either the bearer token (`authorization`) or `x-api-key` — both security
schemes are declared on every v1 operation.

## Step 1 — discover the fields

`POST /reporting/v1/custom_report_fields`

Returns the dimensions and metrics this account may request. Do not assume a field exists
because it existed for another account; entitlements differ.

## Step 2 — discover the filters

- `POST /reporting/v1/custom_report_filters` — the standard filter set
- `POST /reporting/v1/custom_report_single_filter` — values for one filter
- `POST /reporting/v1/custom_report_advanced_filters` — the advanced filter surface

## Step 3 — run it

- `POST /reporting/v1/custom_report` — returns the result inline
- `POST /reporting/v1/custom_report/export` — returns an export, for anything large or destined
  for a data lake or BI tool

## Other families

Swap the prefix; the four-step shape is identical:

| Family | Fields | Filters | Run | Export |
| --- | --- | --- | --- | --- |
| Historical pricing | `pricing_intelligence_report_fields` | `pricing_intelligence_filters` | `pricing_intelligence` | `pricing_intelligence/export` |
| Shoppable recipe | `shoppable_recipe_report_fields` | `shoppable_recipe_report_filters` | `shoppable_recipe_report` | `shoppable_recipe_report/export` |

## Errors

The spec declares only `200` and `422` on the v1 operations, and `200`/`400`/`500` on
`/reporting/authenticate`. That is a gap, not a guarantee — expect undeclared `401` and `5xx`
and handle them.

A `422` returns FastAPI's validation shape:

```json
{ "detail": [ { "loc": ["body", "fields", 0], "msg": "...", "type": "..." } ] }
```

Read `detail[].loc` to find the offending field path, then re-run step 1 or 2 rather than
guessing a correction.

## Do not

- Do not construct a report body from field names you saw in documentation or another account.
- Do not reuse a Commerce API key or token against `/reporting/*`.
- Do not poll `/export` in a tight loop; no rate limits are published for this API, which means
  the caps are unknown, not absent.
