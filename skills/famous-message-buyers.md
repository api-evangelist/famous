---
name: famous-message-buyers
description: Send a templated message to a seller's buyers — preview the recipients first, because the send is not idempotent and goes to real customers.
api: Spring Seller API
base_url: https://api.teespring.com/
operations:
  - postV1AuthTokens
  - getSellerV1MessagesVariables
  - getSellerV1MessagesRecipients
  - postSellerV1MessagesSend
generated: '2026-08-13'
method: generated
source: openapi/famous-spring-api-swagger.json
---

# Message Spring buyers

> **Human in the loop is REQUIRED.** `postSellerV1MessagesSend` delivers real email to real
> customers, and the API supports **no idempotency key**. A retry after a network timeout can
> double-send to the entire audience. An agent must not call step 4 without explicit human
> confirmation of the resolved recipient list and the final copy.

## Steps

1. **Authenticate.** See `famous-authenticate-spring-seller`.

2. **Read the available template variables** — `getSellerV1MessagesVariables`

   `GET /seller/v1/messages/variables`

   This is the only seller operation that declares **no parameters at all** — not even
   `access_token`. Fetch it first and build your copy only from the variables it returns; a
   variable that is not on this list will not substitute.

3. **Resolve the audience before sending** — `getSellerV1MessagesRecipients`

   `GET /seller/v1/messages/recipients?access_token=<token>&slug=<campaign-slug>`

   Optional filters are `slug` (a campaign) and `lookup_number` (an order). With neither, you are
   asking about the seller's whole buyer base — confirm that is what you meant.

   Show the resolved count and scope to a human. Get an explicit yes.

4. **Send** — `postSellerV1MessagesSend`

   `POST /seller/v1/messages/send`

   **Form data**, not JSON. All four fields are required:

   | field | notes |
   |---|---|
   | `access_token` | the seller token — sent in the body here, not the query string |
   | `email_type_id` | the message/template type |
   | `subject` | subject line |
   | `content` | body, using only variables from step 2 |

5. **Record the send yourself.** There is no send-history endpoint and no idempotency key. Persist
   the timestamp, the `x-request-id` from the response headers, the resolved recipient scope and a
   hash of the content. That record is the only protection against a duplicate send.

## Failure handling

- **On a timeout or a connection reset, do NOT retry automatically.** You cannot tell a failed send
  from a successful one whose response was lost, and the API gives you no way to ask. Escalate to a
  human.
- Errors arrive as `{"error":"..."}` or `{"error":{"message":"..."}}` — handle both shapes.
- A 404 here may mean an unaccepted parameter value rather than a missing resource; read the
  message before concluding the endpoint is gone.
- No `Retry-After` header is ever returned, so there is no server-supplied backoff to honour.
