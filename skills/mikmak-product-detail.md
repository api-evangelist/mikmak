---
name: mikmak-product-detail
description: >-
  Fetch what a product IS — title, brand, description, packaging and imagery — from MikMak's
  catalog for a given location, without pulling retailer availability.
api: MikMak Headless Commerce API (v1)
base: https://api.mikmak.ai
operations:
  - GET /commerce/v1/search/products
  - GET /commerce/v1/products/{id}
mcp_tools:
  - search_products
  - get_products
generated: '2026-08-12'
method: generated
source: openapi/mikmak-commerce-api-openapi.yml, https://docs.mikmak.ai/docs/tools-1
---

# Get product details

Use this when the user wants to know **what** something is, not **where** to buy it. It is the
cheaper half of the where-to-buy flow and returns no retailer list.

## Step 1 — resolve the id (skip if you already have a GTIN)

`GET /commerce/v1/search/products` — see `mikmak-where-to-buy.md`. Same rules: exact user words,
required ISO country, one call, wait for the user when there are several hits.

## Step 2 — fetch the metadata

`GET /commerce/v1/products/{id}` (MCP: `get_products`)

- `{id}` — one GTIN or several comma-separated.
- `idType` — `gtin` (default), `mpn`, `group`, `product_id` or `matching_id` on the REST
  surface. The MCP tool's `id_type` uses the uppercase set: `DEFAULT`, `GTIN`, `MPN`,
  `PRODUCT_ID`, `MODEL_NAME`, `ACCOUNT_PRODUCT_UID`. Omit it unless the id is not a GTIN.
- A location is still required, under the same exactly-one-of rule — product copy, imagery and
  packaging vary by market.

Read `ProductsResponse`: `location` and `products[]` of `CatalogProduct` — `id`, `ids`, `gtin`,
`gtins`, `brand`, `title`, `subtitle`, `description`, `packaging`, `imageUrl`,
`thumbnailImageUrl`, `review`.

Note `products[]` here is `CatalogProduct`, which is **not** the same schema as the
`ProductsSearchProduct` you got back from search. They differ. Normalise rather than assuming
one shape.

## When to use the other operations instead

| You need | Use |
| --- | --- |
| Retailers, price, stock | `GET /commerce/v1/availabilities/{id}` |
| A whole basket at one store | `GET /commerce/v1/availabilities/cart/{id}` |
| Online + local offers for a model | `GET /commerce/v1/productcatalog/offers/models/{id}` |
| Facets to refine a search | `GET /commerce/v1/products/facet` |

The last three have **no MCP tool** — they are REST-only. See
`mcp/mikmak-tool-crosswalk.yml`.

## Errors

Same table as `mikmak-where-to-buy.md`. The one to expect here is 404 when a GTIN is valid
globally but absent from this experience's catalog — re-resolve through search rather than
retrying.
