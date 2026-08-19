---
name: famous-reconcile-seller-orders
description: Pull a seller's orders for a date range and state, page through them, and tie each order back to its campaign.
api: Spring Seller API
base_url: https://api.teespring.com/
operations:
  - postV1AuthTokens
  - getSellerV1Orders
  - getSellerV1OrdersForBuyerFromOrderLookupNumber
  - getSellerV1Campaigns
  - getSellerV1Payouts
generated: '2026-08-13'
method: generated
source: openapi/famous-spring-api-swagger.json
---

# Reconcile Spring seller orders

Pull the orders behind a period's revenue and attribute them to campaigns and payouts.

## Steps

1. **Authenticate.** Follow `famous-authenticate-spring-seller` to get an `access_token`.

2. **List the campaigns you care about** — `getSellerV1Campaigns`

   `GET /seller/v1/campaigns?access_token=<token>&states=active&per_page=100&page=1`

   Campaign *roots* come back, not campaign runs. The spec is explicit: a root is
   *"groupings of individual campaign runs for a single design"*, and every relaunch adds another
   campaign to the same root. Keep the root `slug` and, if you need the runs,
   `getSellerV1CampaignsSlug` with `history=true`.

   Valid `states`: `deleted, draft, active, suspended, success, failed, archive, redirect, hidden`
   (comma-separated). These are documented in prose only — the spec declares no `enum`, so nothing
   validates your value client-side.

3. **Pull the orders** — `getSellerV1Orders`

   `GET /seller/v1/orders?access_token=<token>&start_date=2026-01-01&end_date=2026-01-31&states=placed,charged&per_page=100&page=1`

   - `start_date` / `end_date` are `YYYY-MM-DD`.
   - `states` defaults to `placed,charged`. The full set is
     `failed, cancelled_and_refunded, cancelled, initialized, placed, charged`. **Ask for
     `cancelled_and_refunded` explicitly** if you are reconciling net revenue — the default hides
     refunds.
   - `campaign_ids` takes a comma-separated list and defaults to `*`.

4. **Page to exhaustion.** Increment `page` until a page comes back short. There is no documented
   default `per_page`, no documented maximum, and the spec declares no response schema, so the
   page-metadata field names are unknown until you see a real response — count the returned records
   rather than trusting a `total` you have not verified.

5. **Follow a buyer** — `getSellerV1OrdersForBuyerFromOrderLookupNumber`

   `GET /seller/v1/orders/for_buyer_from_order/{lookup_number}?access_token=<token>`

   Given one order's lookup number, this returns every order that buyer placed. Use it to collapse
   repeat customers before you count them as distinct.

6. **Tie out to money** — `getSellerV1Payouts`

   `GET /seller/v1/payouts?access_token=<token>&per_page=100&page=1`

   Returns payout *requests* as well as actual payouts. Orders and payouts do not settle on the
   same clock, so reconcile them as two ledgers, not one.

## What will bite you

- **Everything here is read-only** — there is no write path into orders, so a re-run is safe.
- **No rate limits are published and no rate-limit headers are returned.** When paging deep, back
  off exponentially on any non-JSON response; you cannot tell throttling from routing by status
  code alone.
- **`slug` is type-inconsistent** across the API: a string on the seller surface, an `int32` on
  `GET /v1/campaigns/{slug}`. Do not reuse one generated model for both.
- **Cache pressure.** `getSellerV1Summary` accepts `flush_cache=true`; the order endpoints do not.
  If a summary and your order total disagree, flush the summary before assuming the orders are
  wrong.
