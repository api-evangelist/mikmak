---
name: mikmak-where-to-buy
description: >-
  Answer "where can I buy X near me" against MikMak's retailer network — resolve a product name
  to a GTIN, then return ranked retailers with price, stock, delivery mode and a click-through
  URL for a specific location.
api: MikMak Headless Commerce API (v1)
base: https://api.mikmak.ai
operations:
  - GET /commerce/v1/search/products
  - GET /commerce/v1/availabilities/{id}
mcp_tools:
  - search_products
  - search_availabilities
generated: '2026-08-12'
method: generated
source: openapi/mikmak-commerce-api-openapi.yml, https://docs.mikmak.ai/docs/tools-1
---

# Where to buy a product

This is MikMak's marquee flow. It always runs in two steps, and the second step must never be
guessed at.

## Before you start

You need three things MikMak issues at onboarding and does not publish: an API key
(`x-api-key`), an experience id (`x-wtb-id`), and — on the MCP path — an Auth0 JWT minted for
the audience MikMak gave you. See `authentication/mikmak-authentication.yml`.

You also need a location from the user. **Do not invent one.** If the user has not given a
postal code, a city, or coordinates, ask.

## Step 1 — resolve the product

`GET /commerce/v1/search/products` (MCP: `search_products`)

- Pass the user's **exact words** as the query. Do not rephrase, expand, or add brand names the
  user did not say.
- `country` is required and must be ISO 3166-1 alpha-2. Infer it only from an explicit clue in
  the message — a place name, a ZIP code, "in the UK". If there is no clue, **ask**. A wrong
  country silently returns the wrong catalog.
- Call it **once**. If it returns nothing, do not retry with different keywords.

Read `ProductsSearchResponse`: `offset`, `limit`, `total`, `products[]`. Each product carries
`id`, `gtin`, `brand`, `title`, `packaging` and images.

Then branch on the count:

| Results | Do this |
| --- | --- |
| Exactly one | Continue to step 2 with that `id`. |
| Zero | Tell the user, show what you searched, stop. |
| Many | Show the list and **wait for the user to pick**. Do not auto-select the first hit. |

## Step 2 — find where to buy it

`GET /commerce/v1/availabilities/{id}` (MCP: `search_availabilities`)

`{id}` is the GTIN from step 1, or several comma-separated. Never a GTIN you constructed.

Supply **exactly one** location mode:

- postal — `postal_code` + `country`, optionally refined with `state` and `city`
- coordinates — `latitude` (−90…90) + `longitude` (−180…180)

Sending both is rejected: *"Do not mix location methods. Use either (latitude + longitude) OR
(postal_code + country), not both."*

Only send optional filters the user actually asked for — `max_distance` (kilometres),
`max_stores`, `delivery_modes` (`HomeDelivery`, `Drive`, `CollectInStore`, `Brick`),
`include_out_of_stock`, `retailer_ids`.

Read `AvailabilitiesResponse`: `location` (the place MikMak actually resolved — check it against
what the user asked for), `products[]`, `availabilities[]` with price, stock status and
`clickUrl`, plus an optional `voucher`, and `sessionId`.

**Send the shopper to `clickUrl`.** There is a `doNotTrackClickUrl` twin on every offer — use it
when the user has not consented to tracking, and only then.

Distances in the response are metres. `max_distance` on the request is kilometres. They do not
match; convert.

## Errors

| Status | Meaning | What to do |
| --- | --- | --- |
| 400 | Missing/malformed parameters or bad location data | Re-check the one-of location rule |
| 401 | Missing or expired key/token | Refresh the token, retry **once**, never in a loop |
| 403 | Authenticated but not permitted — country out of scope, product not in the catalog, retailer disabled | Do not retry. Tell the user this experience does not cover that country or product |
| 404 | Product, experience or continuation token not found | Re-resolve through step 1 |
| 429 | Rate cap hit | Honour `Retry-After`; watch `X-RateLimit-Remaining` on successes |

On the MCP surface, two upstream 400s come back as **normal text with `isError: false`** — "no
product catalog configured for this experience" and "requested country out of scope". Read them
as guidance, not failure.

## Do not

- Do not invent a GTIN, a postal code, or a country.
- Do not skip step 1 because the user said something that looks like a product code.
- Do not page the search results; pagination is server-controlled and not exposed.
- Do not put the API key anywhere a browser can see it.
