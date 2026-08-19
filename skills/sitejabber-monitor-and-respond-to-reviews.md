---
name: sitejabber-monitor-and-respond-to-reviews
description: Pull new SmartCustomer reviews for a business, triage them, and reply publicly or privately.
api: SmartCustomer (Sitejabber) Business API
base: https://api.smartcustomer.com/v1
operations:
- getBusiness
- getBusinessReviews
- getBusinessReview
- addReviewComment
- sendResolutionMessage
- flagReview
generated: '2026-08-13'
method: generated
source: openapi/sitejabber-business-api-openapi.yml + https://api.sitejabber.com/
---

# Monitor and respond to reviews

Poll a business's reviews on a schedule, then reply in public or reach the reviewer in private.

## Before you start

Both credentials are required on every call after login:

1. `POST /login?client_token={apiKey}` with form fields `email` and `password` (`login`).
2. Read `token` from the Login object and send it as a `user_token` request header from then on.
3. Keep `client_token={apiKey}` in the query string of every call as well.

The token lasts about six months and the response carries an `expire` timestamp — store it. Calling
`login` again invalidates the previous token, so do not re-login on every run or you will evict
another client sharing the account.

## Steps

1. **Confirm the business.** `getBusiness` — `GET /businesses/{business}` where `{business}` is the
   domain without a scheme (`yourdomain.com`). Read `averageRating` and `numReviews` for the
   baseline. Error `301` means the display address is wrong.
2. **Pull the new window.** `getBusinessReviews` — `GET /businesses/{business}/reviews` with
   `date_from` set to your last run (`yyyy-mm-dd hh:mm:ss`, time optional), `count=100`,
   `order=DESC`. Page with `start` until a page returns fewer than `count` rows; there is no total
   count and no cursor, so short page is the only end-of-data signal.
   - Add `unpublished=1` if you also want reviews awaiting publication.
   - Add `datasources=google,trustpilot,yelp,...` to widen to aggregated third-party sources.
3. **Triage each review.** `rating` is an array of Rating objects — read the entry whose `type` is
   `overall`. `author` is a User object with a first-initial-only `lastName`; treat it as pseudonymous.
4. **Reply in public.** `addReviewComment` — `POST /businesses/{business}/review/comments/add` with
   `review_no` and `text`, form-encoded. This is publicly visible.
5. **Reach the reviewer in private.** `sendResolutionMessage` —
   `POST /businesses/{business}/resolution/send` with `review_no`, `username` and `body`. This
   contacts a real person; get human approval before an agent sends it.
6. **Flag policy violations only.** `flagReview` — `POST /businesses/{business}/reviews/flag` with
   `review_no`, `reason` and `message`. `reason` must be one of `language`, `personal_info`,
   `personal_attack`, `second_hand`, `another_biz`, `other`. Flag on policy grounds, never because a
   review is negative.

## Rules

- **Check the body, not the status.** Application errors come back as HTTP 200 with
  `success: false`, `status: "ERROR"`, `errorCode` and `errorReason`.
- **Body error codes are not HTTP codes.** `errorCode: 404` means "can vote only once every 24
  hours". `401`, `402`, `403` are product/image/review-not-found. See
  `errors/sitejabber-problem-types.yml`.
- **Rate limits: 1000 calls/hour per user token, and never more than 10 calls in any 10 seconds.**
  Exhaustion returns HTTP 429. No `Retry-After` or `RateLimit-*` header is published, so pace
  yourself — roughly one call per second — rather than reacting to a header that will not arrive.
- **`count` is capped at 100.** Exceeding it returns error `303`.
- **No idempotency key exists anywhere on this API.** Do not blind-retry a write after a timeout.
- On `207` (token expired) or `204` (invalid session), re-run `login` once and retry — not in a loop.
