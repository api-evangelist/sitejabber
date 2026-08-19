---
name: sitejabber-sync-product-catalog-and-reviews
description: Keep a product catalog in sync with SmartCustomer and pull product reviews, Q&A and rating stats for storefront display.
api: SmartCustomer (Sitejabber) Business API
base: https://api.smartcustomer.com/v1
operations:
- getProducts
- addProduct
- removeProduct
- addProductImage
- getProductReviews
- getProductReviewsStats
- getTopRatedProduct
- getProductQuestions
generated: '2026-08-13'
method: generated
source: openapi/sitejabber-business-api-openapi.yml + https://api.sitejabber.com/
---

# Sync the product catalog and pull product reviews

## Sync the catalog

1. **Read what is there.** `getProducts` — `GET /businesses/{business}/products`. Filter by `sku`,
   `gtin`, `mpn`, `product_id`, `brand`, `categories`, `q`, `price_range` or `reviews_range`. Page
   with `start`/`count` (`count` max 100).
2. **Add or update.** `addProduct` — `POST /businesses/{business}/products/add`. Required: `sku`,
   `title`. Optional: `item_group` (groups variants — this is what makes `item_group` review
   rollups work later), `gtin`, `mpn`, `description`, `brand`, `categories` (comma-separated),
   `currency`, `price`, `retail_price`, `product_link`, `image_url`, `attributes`.
3. **Extra images.** `addProductImage` — `POST /businesses/{business}/product/images/add` with `sku`
   and `image` (a URL). Error `311` means the upload failed.
4. **Remove.** `removeProduct` — `POST /businesses/{business}/products/remove` with `product_id`.
   Note the asymmetry: you add by `sku` but remove by `product_id`, so resolve the ID with
   `getProducts` first.

## Pull reviews for display

- `getProductReviews` — `GET /businesses/{business}/product/reviews`. Narrow with `sku`, or with
  `item_group` to roll up a variant family (**when `item_group` is set, `sku` is ignored**). Also
  `brand`, `categories`, `q`, `ratings` (e.g. `4,5`), `only_photos=1`, `price_range`,
  `reviews_range`, `date_from`/`date_to`.
  - `order` here takes `created` (most recent) or `relevant` (most relevant) — **not** the `ASC`/`DESC`
    the rest of the API uses. Passing `DESC` here silently gets you the default.
- `getProductReviewsStats` — `GET /businesses/{business}/product/reviews/stats` for aggregates by
  `sku`, `categories`, `item_group` or `brand`. Use this for the star summary rather than counting
  reviews client-side.
- `getTopRatedProduct` — `GET /businesses/{business}/product/reviews/top-rated`, optionally
  `category_name` and `currency`.
- `getProductQuestions` — `GET /businesses/{business}/product/questions`, filter by `sku` and
  `filter` (`PENDING`/`PUBLISHED`/`REMOVED`, default is non-removed).

## Rules

- Application errors are HTTP 200 with `success: false`; check `errorCode`. `401` = product does not
  exist, `402` = image does not exist, `403` = product review does not exist, `303` = a paging limit
  was exceeded.
- `count` maxes at 100 on every list call. There is no total and no cursor — page until a page comes
  back short.
- Rate limits: 1000/hour per user token, max 10 calls per 10 seconds. A full catalog sync must be
  paced; at 100 products per page and 10 calls per 10s a large catalog will take real time.
- No idempotency key: a retried `addProduct` after a timeout risks a duplicate. Read back with
  `getProducts?sku=` before retrying.
- Do not use `addProductReview` or `addProductReviewVote` to write reviews or votes on a customer's
  behalf. Those are deliberately excluded from this skill.
