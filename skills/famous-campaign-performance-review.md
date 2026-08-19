---
name: famous-campaign-performance-review
description: Build a seller performance review from the Spring dashboard, period summary, campaign rankings, promotions and volume-discount tier.
api: Spring Seller API
base_url: https://api.teespring.com/
operations:
  - postV1AuthTokens
  - getSellerV1Dashboard
  - getSellerV1Summary
  - getSellerV1Campaigns
  - getSellerV1CampaignsSlug
  - getSellerV1Promotions
  - getSellerV1DashboardVolumeDiscount
generated: '2026-08-13'
method: generated
source: openapi/famous-spring-api-swagger.json
---

# Review Spring campaign performance

Assemble the same picture the Spring seller dashboard shows, plus the rankings the UI does not
expose directly.

## Steps

1. **Authenticate.** See `famous-authenticate-spring-seller`.

2. **Take the whole dashboard in one call** — `getSellerV1Dashboard`

   `GET /seller/v1/dashboard?access_token=<token>`

   The spec describes it as *"All the information used to populate the Seller Dashboard, pulled
   from exactly the same source."* Start here — it is one round trip and it is authoritative.

3. **Get the period numbers** — `getSellerV1Summary`

   `GET /seller/v1/summary?access_token=<token>&period=thirty_days`

   `period` is **required** and accepts exactly: `today`, `yesterday`, `week`, `month`,
   `seven_days`, `thirty_days`. Anything else returns **HTTP 404** with
   `{"error":{"message":"Unsupported period. Valid periods include: ..."}}`.

   Two optional parameters matter:
   - `slug` narrows the summary to one campaign root.
   - `flush_cache=true` forces a fresh computation. Use it once at the start of a review, not on
     every call.

4. **Rank the catalogue** — `getSellerV1Campaigns`

   `GET /seller/v1/campaigns?access_token=<token>&sort_by_sales=desc&per_page=100&page=1`

   Sort options are `sort_by_sales`, `sort_by_profit` (each `asc` or `desc`) and `recency`. Do not
   send two sorts at once — precedence is undocumented. `search` filters by title or description.

5. **Open the winners** — `getSellerV1CampaignsSlug`

   `GET /seller/v1/campaigns/{slug}?access_token=<token>&history=true`

   Returns the campaign root plus every run attached to it. `history=true` includes previous runs —
   essential for a relaunched design, because sales are spread across runs under one root.
   `getvars=true` adds the GET-variable data.

6. **Add the commercial context**

   - `getSellerV1Promotions` — `GET /seller/v1/promotions?access_token=<token>` for the seller's
     active promo codes. Discounts explain margin gaps that the campaign ranking alone will not.
   - `getSellerV1DashboardVolumeDiscount` — `GET /seller/v1/dashboard/volume_discount?access_token=<token>`
     for the volume-discount tier the seller currently qualifies for. Unit economics move with this
     tier, so a period-over-period profit change may be a tier change rather than a demand change.

## What will bite you

- **Campaign roots vs campaigns.** Ranking roots by sales and then quoting a single run's numbers
  is the most common way to get this wrong. Always resolve the root before attributing revenue.
- **Cached summaries.** If a summary disagrees with a campaign ranking, re-request the summary with
  `flush_cache=true` before investigating further.
- **No response schemas.** The spec declares none, so field names must be read from a real
  response. Do not hard-code a field you have not seen.
- **Read-only.** Every operation in this skill is a GET; a re-run is always safe.
