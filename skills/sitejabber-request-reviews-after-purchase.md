---
name: sitejabber-request-reviews-after-purchase
description: Ask a customer for a company review or a product review after an order, and cancel outstanding requests.
api: SmartCustomer (Sitejabber) Business API
base: https://api.smartcustomer.com/v1
operations:
- createReviewRequest
- removeReviewRequest
- createProductReviewRequest
- removeProductReviewRequest
- getWriteReviewLink
generated: '2026-08-13'
method: generated
source: openapi/sitejabber-business-api-openapi.yml + https://api.sitejabber.com/
---

# Request reviews after a purchase

Two separate flows: a **company** review request and a **product** review request. They use
different endpoints and different required fields.

## Company review request

`createReviewRequest` — `POST /businesses/{business}/review/request/add`, form-encoded.

Required: `order_id`, `order_date` (`yyyy-mm-dd`), and **either** `email` **or** `phone` — the
reference marks both required, noting each is required only when the other is absent. Supplying
`phone` is what makes it an SMS request.

Optional: `first_name`, `last_name`, `labels` (comma-separated, attached to the review once the
customer completes it — this is how you segment reviews later), `location`, `language`,
`product_skus`, `return_link`.

Set `return_link=1` to get the completion link back **instead of** SmartCustomer sending the email.
Use that when you want to place the link in your own transactional mail.

## Product review request

`createProductReviewRequest` — `POST /businesses/{business}/product/review/request/add`.

Required: `email`, `product_skus` (comma-separated), `order_id`, `order_date`. Same `return_link`
behavior. The SKUs must already exist in the catalog — add them first with `addProduct`
(`POST /businesses/{business}/products/add`, requires `sku` and `title`) or you will get error `401`
(product does not exist).

## Cancelling

- `removeReviewRequest` — `POST /businesses/{business}/review/request/remove` with `email`. Omit
  `order_id` and **every** outstanding request for that email is removed.
- `removeProductReviewRequest` — `POST /businesses/{business}/product/review/request/remove` with
  `email`; optional `product_sku` and `order_id` narrow it. Omit both and all of them go.

## Partner links

`getWriteReviewLink` — `GET /partners/{business}/write-link/get` returns a Partner object
(`url`, `hash`, `expire`) for a review page you can hand to a customer directly. `getEditReviewLink`
does the same for editing an existing review (pass `email`); the user still has to log in to edit.

## Rules

- **These operations contact real people by email or SMS.** Get explicit human authorization before
  an agent triggers them in bulk.
- **There is no idempotency key.** If a request times out, do not retry blindly — you may double-send
  to the customer. Error `304` ("Email already exists") is the only guard the API offers, and you
  cannot rely on it firing.
- Application errors are HTTP 200 with `success: false`. Read `errorCode`.
- Watch `309` (review request removal error) and `310` (product review request removal error) on the
  cancel calls.
- Rate limits: 1000/hour per user token, max 10 calls per 10 seconds. Batch sends need pacing.
- Never fabricate an `order_id` or `order_date` to force a request through; the request is an
  assertion that a real purchase happened.
