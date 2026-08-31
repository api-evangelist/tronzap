---
name: tronzap-aml-screen-tron-address
description: >-
  Screen a TRON address or transaction hash for AML risk through the TronZap account API —
  list the available screening services, create a check, and poll it to a result.
api: TronZap REST API
base_url: https://api.tronzap.com/v1/
auth: Bearer token + X-Signature (SHA-256 of raw body + API secret)
docs: https://docs.tronzap.com/api/aml-checks.html
operations:
  - POST /v1/aml-checks
  - POST /v1/aml-checks/new
  - POST /v1/aml-checks/check
  - POST /v1/aml-checks/history
generated: '2026-08-30'
method: generated
source: >-
  Grounded in https://docs.tronzap.com/api/aml-checks.html, /aml-check-create.html,
  /aml-check-status.html, /aml-checks-history.html and /api/error-codes.html (fetched
  2026-08-30). No OpenAPI is published; operations are quoted from the HTML reference.
---

# Screen a TRON address or transaction for AML risk

TronZap sells blockchain AML screening alongside its energy product. Checks are billed, run
asynchronously, and are keyed on either a TRON address or a transaction hash.

## Before you start

- Same auth as every other account endpoint: `Authorization: Bearer <token>`,
  `X-Signature: sha256(raw_body + secret)`, `Content-Type: application/json`. All POST.
- Errors come back as HTTP 200 with a non-zero `code`. Read the body.
- Screening is **billed and not cancellable** once created. Price it against
  `POST /v1/aml-checks` before you create anything in a loop.
- TronZap does not name the screening provider or the sanctions lists behind the result, so
  treat the risk output as one signal, not as a compliance determination.

## Steps

1. **List what is available and what it costs.** `POST /v1/aml-checks` returns the active AML
   services and their pricing. Do this once per session, not per address.

2. **Create the check.** `POST /v1/aml-checks/new` with either an `address` or a `hash`.
   - Supplying a `hash` without the fields the service needs returns code `2` with sub-key
     `invalid_service_or_params.hash` ("Hash is required for hash checks").
   - A malformed address returns code `10` / `invalid_tron_address`.
   - An unsupported network returns `invalid_service_or_params.network`.
   - `direction` is validated; a bad value returns `invalid_service_or_params.direction`.

3. **Poll for the result.** `POST /v1/aml-checks/check` until the status is terminal. States
   are `pending → processing → completed | failed`. A check that cannot be found returns code
   `30` / `aml_check_not_found` — re-create it or contact support with the `request_id`.

4. **Reconcile in bulk.** `POST /v1/aml-checks/history` paginates every check you have made:
   `page` (default 1, minimum 1) and `per_page` (default 10, maximum 50), with an optional
   `status` filter taking `pending`, `processing`, `completed` or `failed`. There is no
   cursor and no documented total-count field, so keep requesting pages until one comes back
   short. Bad values return `invalid_service_or_params.page` / `.per_page` / `.status`.

## Retry safety

There is no idempotency key. A retried create after a timeout will bill a second screening.
Call `POST /v1/aml-checks/history` filtered to `pending` before retrying.
