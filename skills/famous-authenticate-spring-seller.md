---
name: famous-authenticate-spring-seller
description: Mint and use a 24-hour Spring Seller API access token, and handle its expiry.
api: Spring Seller API
base_url: https://api.teespring.com/
operations:
  - postV1AuthTokens
  - getV1UsersMe
generated: '2026-08-13'
method: generated
source: openapi/famous-spring-api-swagger.json + https://api.teespring.com/docs
---

# Authenticate against the Spring Seller API

Every `seller/v1/*` operation needs an `access_token`. This skill mints one and proves it works.

## Before you start

You need an **`app_id`**. There is no self-serve key page anywhere on api.teespring.com, spri.ng,
amazecommerce.com or amaze.co — Spring issues it manually. The docs say only: *"Obtain an app_id
from Teespring."* If you do not have one, stop here; no operation below will work.

You also need the seller's **account email and password**. The token exchange takes the primary
login credential, not a revocable API key. Treat this as a credential-handling task, not a routine
API call.

## Steps

1. **Mint the token** — `postV1AuthTokens`

   `POST https://api.teespring.com/v1/auth-tokens`

   Send `app_id`, `email` and `password` as **form data**, not JSON. The operation declares
   `in: formData` for all three; a JSON body will not be read.

   A success is **201**, not 200. Read `access_token` from the response.

2. **Verify the token** — `getV1UsersMe`

   `GET https://api.teespring.com/v1/users/me?access_token=<token>`

   If this returns the authenticated user, the token is live.

3. **Use it on every call.** `access_token` is a **query parameter**, not an `Authorization`
   header. The one exception is `postSellerV1MessagesSend`, which takes it as form data.

## Expiry and reuse

The token lasts **24 hours**. Re-calling `postV1AuthTokens` does **not** mint a new one — the docs
say it *"will always give you the current one"*. So:

- Cache the token for the day; do not re-authenticate per request.
- On a 4xx that mentions a missing or invalid token, re-run step 1 once, then retry.
- There is no refresh token and no revocation endpoint. Rotating a compromised token means changing
  the seller's password.

## What will bite you

- **Credentials in the URL.** `access_token` travels in the query string, so it lands in proxy
  logs, browser history and `Referer` headers. Never log the full request URL.
- **No scopes.** One token grants everything the seller has, including payouts and buyer PII. There
  is no read-only credential.
- **Error shapes are inconsistent.** A missing token can come back as `{"error":"access_token is
  missing, period is missing"}` (string) or `{"error":{"message":"app_id is missing"}}` (object).
  Handle both — see `errors/famous-problem-types.yml`.
- **404 does not mean "not found".** An invalid enum value returns 404. An unknown path returns
  301. Assert `content-type: application/json` before parsing anything.
