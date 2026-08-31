---
name: reefapi-product-intelligence
description: Assemble a full picture of one retail product — detail, live offers, reviews and review sentiment, plus competing listings — from Amazon via ReefAPI.
api: ReefAPI REST API
generated: '2026-08-31'
method: generated
source: openapi/reefapi-openapi.json (operationIds verified against the spec), https://reefapi.com/docs/amazon.md
operations:
  - amazon_search
  - amazon_product_detail
  - amazon_product_offers
  - amazon_product_reviews
  - amazon_product_related
  - amazon_products_batch
  - amazon_gtin_to_asin
  - amazon_bestsellers
---

# ReefAPI: product intelligence for one ASIN

Every call is `POST https://api.reefapi.com/amazon/v1/<action>` with `x-api-key` and a JSON
body. Every response is `{ ok, data, meta, error }` — branch on `ok`.

## Steps

1. **Resolve the product.** If you have keywords, call `amazon_search`
   (`POST /amazon/v1/search`) with `{ "query": "...", "marketplace": "com" }` and take the
   `asin` off the best result. If you have a barcode instead, call `amazon_gtin_to_asin`
   (`POST /amazon/v1/gtin-to-asin`) to convert a UPC/EAN/ISBN to an ASIN.
2. **Get the product record.** `amazon_product_detail`
   (`POST /amazon/v1/product/detail`, 2 credits) with `{ "asin": "...", "marketplace": "com" }`.
   Returns title, price, currency, rating, brand, features, images, variations, tech specs,
   A+ content, comparable products, coupon/campaign badges and barcodes — plus the top-8
   inline reviews as a **sibling** of `product`, not nested inside it.
3. **Get live commercial state.** `amazon_product_offers`
   (`POST /amazon/v1/product/offers`, 2 credits) for the pinned buy-box plus every competing
   offer with price, seller, condition, ships-from, delivery and Prime flag. This is the
   call that answers "who is winning the buy box and at what price".
4. **Get depth on sentiment.** `amazon_product_reviews`
   (`POST /amazon/v1/product/reviews`, 1 credit). Pass `marketplaces: [...]` instead of
   `marketplace` to merge and de-duplicate reviews across several Amazon sites for more
   volume than any single site carries. The response includes `review_insights` — Amazon's
   own "Customers say" aspect summary — which is `null` when the product has none.
5. **Widen to the competitive set.** `amazon_product_related`
   (`POST /amazon/v1/product/related`) for recommended and related ASINs, then
   `amazon_products_batch` (`POST /amazon/v1/products_batch`, up to 15 ASINs per call) to
   price the whole set in one request instead of N. For category context use
   `amazon_bestsellers` (`POST /amazon/v1/bestsellers`) with a department category slug.

## Rules

- **Batch before you loop.** `amazon_products_batch` takes up to 15 ASINs per call; issuing
  15 single `product/detail` calls costs more credits and burns your ~5 req/s budget.
- **`marketplace` is an enum, not free text.** 19 verified values (`com`, `co.uk`, `de`,
  `fr`, `it`, `es`, `co.jp`, `in`, `ca`, `com.au`, `com.mx`, `com.br`, `com.tr`, `ae`, `nl`,
  `se`, `pl`, `sg`, `sa`); other sites are normalised by the engine. It sets both language
  and currency — always report which marketplace a price came from.
- **Read the field caveats.** The engine documents which filters are best-effort
  (`prime`, `brand`, `min_rating` below 4 stars) rather than strict. Do not present a
  best-effort filter's output as an exact filter.
- **Handle `TARGET_BLOCKED` and `PARSE_ERROR`.** These are the real failure modes of a
  scraping-backed engine: `TARGET_BLOCKED` (502) is usually retryable, `PARSE_ERROR` (502)
  means the upstream page changed shape and needs a provider fix — do not retry it in a
  loop. Both cost 0 credits.
- **Prices are point-in-time.** Every call fetches live; nothing is cached. Timestamp any
  price you report.

## Cost

Detail and offers are 2 credits each, search/reviews/batch 1 credit. Read the exact price
from the endpoint's docs page or from `get_action_schema` over MCP before a bulk run.
