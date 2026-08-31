---
name: tronzap-rent-energy-for-usdt-transfer
description: >-
  Rent TRON Energy through the authenticated TronZap account API so a wallet can send a
  USDT (TRC-20) transfer without burning TRX. Covers pricing the transfer, checking balance,
  creating the purchase and polling to delivery.
api: TronZap REST API
base_url: https://api.tronzap.com/v1/
auth: Bearer token + X-Signature (SHA-256 of raw body + API secret)
docs: https://docs.tronzap.com/api/
operations:
  - POST /v1/services
  - POST /v1/estimate-energy
  - POST /v1/calculate
  - POST /v1/balance
  - POST /v1/transaction/new
  - POST /v1/transaction/check
generated: '2026-08-30'
method: generated
source: >-
  Grounded in the endpoints, parameters, statuses and error codes published at
  https://docs.tronzap.com/api/ (fetched 2026-08-30). TronZap publishes no OpenAPI, so every
  operation below is quoted from the HTML reference rather than from a spec; none is invented.
---

# Rent TRON Energy for a USDT (TRC-20) transfer

A standard USDT TRC-20 transfer costs about 13.40 TRX in burned fees when the sending wallet
has no Energy. Renting ~65,000 Energy (or ~131,000 to a wallet that has never held USDT) drops
that to roughly 3 TRX. This skill buys that Energy through the authenticated account API.

## Before you start

- Every call is a **POST**, even the reads. There are no GET endpoints and no query params.
- Every call needs three headers: `Authorization: Bearer <API_TOKEN>`, `X-Signature: <sig>`,
  `Content-Type: application/json`.
- `X-Signature` is `sha256(<the exact raw JSON body you will send> + <API_SECRET>)`, hex.
  Sign the serialized string you actually transmit — re-serializing after signing fails with
  code `1` / key `auth`.
- **Errors arrive as HTTP 200.** Read `code` in the body on every response. Zero is success.
  Branch on `key`, never on the `error` prose.
- **This flow spends real funds and cannot be undone.** There is no cancel, void or refund on
  an account transaction; TronZap states purchases are not refundable. Get human confirmation
  before step 4.

## Steps

1. **Confirm the service is live and learn the bounds.**
   `POST /v1/services` → `result.energy[]` carries `min_amount`, `max_amount`, `price` (per
   1,000 energy) and `price_32k`. Ignore `min_energy` / `max_energy`; they are deprecated
   aliases of `min_amount` / `max_amount`. If the service is unavailable you will get code
   `35` / `service_unavailable` — stop and retry later.

2. **Size the purchase.**
   `POST /v1/estimate-energy` with the transfer you intend to make → `result.amount` is the
   Energy needed and `result.price` the TRX cost. Use `result.amount`, not the deprecated
   `result.energy`. If the destination has never held USDT, expect roughly double.

3. **Price it without committing.**
   `POST /v1/calculate` with `{address, amount, duration: 1, type: "energy"}` →
   `result.price`. This is a true dry run: it creates nothing. `duration` must be `1`; any
   other value returns code `12` / `invalid_duration.duration`.

   Then `POST /v1/balance` and confirm the account can cover `result.price`. Skipping this
   turns into code `6` / `insufficient_funds` at step 4.

4. **Buy.** *(irreversible — confirm with a human first)*
   ```
   POST /v1/transaction/new
   {
     "external_id": "<your own id>",
     "service": "energy",
     "params": {
       "address": "<TRON address, 34 chars, leading T>",
       "amounts": { "energy": 65000 },
       "duration": 1
     }
   }
   ```
   Use `params.amounts` — the provider recommends it as the universal form and resolves
   `params.amounts` → `params.amount` → the deprecated `params.energy_amount` in that order.
   Always send your own `external_id`; it is the only handle you have if the response is lost.

   If the address has never been activated you get code `24` / `address_not_activated`. Either
   set `params.activate_address: true`, or create a separate transaction with
   `service: "activate_address"` first. Code `25` means it was already active — proceed.

5. **Poll to a terminal state.**
   `POST /v1/transaction/check` with the id (or your `external_id`). Statuses run
   `new → pending → success | failed`; `success` and `failed` are terminal. Delegated Energy
   is available for one hour.

## Failure handling

| `key` | code | What to do |
|---|---|---|
| `auth` | 1 | Re-derive the signature over the exact bytes sent. |
| `invalid_service_or_params` | 2 | Read the dot sub-key — it names the offending field. |
| `insufficient_funds` | 6 | Top up, or lower the requested amount. |
| `invalid_tron_address` | 10 | Validate 34 chars / leading `T` before sending. |
| `invalid_energy_amount` | 11 | Re-read the min/max from `POST /v1/services`. |
| `invalid_duration` | 12 | `duration` must be `1`. |
| `address_not_activated` | 24 | Activate first, or pass `params.activate_address: true`. |
| `service_unavailable` | 35 | Back off and retry. |
| `internal_server_error` | 500 | Retry with back-off; keep `request_id` for support. |

## Retry safety

There is **no idempotency key**. A retried `POST /v1/transaction/new` after a network timeout
can create a second paid transaction. Before retrying, call `POST /v1/transaction/check` with
the `external_id` you sent and confirm nothing was created.
