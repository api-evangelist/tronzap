---
name: tronzap-wallet-orders-public-api
description: >-
  Create, track and cancel TRON Energy orders through TronZap's fully public Wallet Orders
  API — no account, no API key, no signature. The lowest-friction agent surface TronZap
  operates, and the only one with a documented cancel window.
api: TronZap Wallet Orders API
base_url: https://api.tronzap.com/v1/
auth: none
docs: https://docs-wallets.tronzap.com/api/
operations:
  - POST /v1/orders/calculate
  - POST /v1/orders
  - POST /v1/orders/check
  - POST /v1/orders/cancel
generated: '2026-08-30'
method: generated
source: >-
  Grounded in https://docs-wallets.tronzap.com/api/ , /api/calculate.html, /api/create-order.html,
  /api/check-order.html, /api/cancel-order.html, /api/order-statuses.html and
  /api/error-codes.html (fetched 2026-08-30), plus a live unauthenticated probe of
  POST /v1/orders/calculate on 2026-08-30 (HTTP 200, x-ratelimit-limit: 1).
---

# Order TRON Energy without an account

TronZap's Orders API is built for non-custodial wallets embedding energy purchase into their
own flow. It is genuinely public: the provider states "No authentication headers are required.
The Orders API is fully public." Only `Content-Type: application/json` and
`Accept: application/json` are needed.

## The hard constraints

- **1 request per second, per IP, across all `/v1/orders*` endpoints.** Exceeding it returns
  HTTP `429` with body `{"code":429,"error":"Too Many Requests"}` and **no `Retry-After`
  header** — back off at least a second and grow exponentially. Confirmed live on 2026-08-30:
  responses carry `x-ratelimit-limit: 1` and `x-ratelimit-remaining: 0`.
- **Errors arrive as HTTP 200** with a non-zero `code`, except rate limiting (429), validation
  (422) and server faults (500). Always read `code`.
- **You cannot hold two active orders for the same `address_from` + `address_to` pair** while
  the first is still `new`. On that failure, call `POST /v1/orders/check` — do not retry the
  create.
- **Pass `referral_code` on every create** if commission matters. It is set once at
  `dash.tronzap.com/referral-earnings` and can never be changed. An order without it is
  attributed to nobody.

## Steps

1. **Quote first.** `POST /v1/orders/calculate`
   ```json
   { "duration": 1, "activation": false, "resources": { "energy": 65000, "bandwidth": 345 } }
   ```
   This is a genuine dry run — the provider states it "does not touch the TRON network", and
   guarantees `result.total` equals the `amount` a subsequent order will require for the same
   inputs. Show `result.total` and `result.currency` (always `TRX`) to the user before
   committing. `duration` must be `1`. At least one of `resources.energy` /
   `resources.bandwidth` must be greater than zero.

2. **Create the order.** *(payable — get user intent first)* `POST /v1/orders`
   ```json
   {
     "referral_code": "YOUR_CODE",
     "address_from": "<payer TRON address>",
     "address_to":   "<recipient TRON address>",
     "resources": { "duration": 1, "energy": 65000 }
   }
   ```
   You get back `order_id` (a ULID), `status: "new"`, `deposit_address`, `amount`, `currency`,
   `expires_at` and `ttl` (typically 3600 seconds).

3. **Tell the user to pay exactly.** The TRX transfer must originate from `address_from` and
   be **exactly** `amount` TRX. Any other sender or any other amount will not match the order
   and the order will expire.

4. **Poll.** `POST /v1/orders/check` with `{ "order_id": "..." }` every 5–10 seconds after the
   user says they have paid. Statuses run
   `new → paid → processing → completed`, with `failed`, `cancelled` and `expired` as the other
   terminal states. Stop polling at any terminal state or when `ttl` reaches 0. Most orders
   complete within 5–10 seconds of payment confirmation. Respect the 1 req/s ceiling — a
   5-second interval is well inside it.

## Cancelling — the one reversible write

`POST /v1/orders/cancel` with `{ "order_id": "..." }` works **only while the order is still in
`new` status** — that is, before a deposit is matched and before `expires_at`. That is the
window: roughly one hour from creation, exactly `ttl` seconds.

> **There is no automatic refund.** The provider is explicit: "Once an order is cancelled, any
> TRX received later for that order is not refunded automatically." Cancel only when you are
> certain the user has not sent the deposit. If in doubt, let the order expire instead — an
> expired order costs the user nothing if they never paid.

Cancelling an order already `paid`, `processing` or terminal returns an error.

## Common failures

| Situation | Cause |
|---|---|
| Invalid `address_from` / `address_to` | Not a TRON address (must start with `T`, 34 chars). |
| `duration` not `1` | Only 1-hour rentals exist. |
| Both energy and bandwidth `0` | Request at least one resource. |
| Amount out of bounds | Below the minimum or above the maximum for this project. |
| Duplicate active order | Use `POST /v1/orders/check` on the existing order. |
| `order_id` not found | The ULID does not exist. |
| HTTP 429 | More than 1 req/s from your IP. Exponential back-off; no `Retry-After` is sent. |
| `referral_code` too long | Maximum 64 characters. |

Log `request_id` from every response — it is the only correlation handle TronZap support asks
for, and there is no client-supplied trace header.
