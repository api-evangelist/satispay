---
name: satispay-refund-a-payment
description: >-
  Reverse a settled Satispay payment. There is no refund endpoint — a refund is a new payment with
  flow=REFUND — and the window that is still open depends on which instrument funded the original.
api: Satispay GBusiness API
base_url: https://authservices.satispay.com/g_business/v1
operations:
- get-the-details-of-a-payment
- create-a-payment
- update-a-payment
generated: '2026-08-26'
method: generated
source: >-
  openapi/satispay-gbusiness-api.json, https://developers.satispay.com/reference/refund,
  https://developers.satispay.com/reference/update-a-payment
---

# Refund a Satispay payment

## The shape to know first

There is no `DELETE` and no `/refunds` endpoint. A refund is **a new payment** that points back at the
original. Refunds also cannot be done from the Dashboard — Satispay states they are only possible via
the API.

## 1. Check the payment is refundable

`GET /payments/{id}` (`get-the-details-of-a-payment`). It must be `ACCEPTED`. If it is still `PENDING`
or `AUTHORIZED`, do not refund — cancel it instead with `PUT /payments/{id}`, `action: CANCEL`.

## 2. Check the window — this is instrument-dependent

| funding instrument | window | partial refunds |
|---|---|---|
| e-money | within **365 days** of payment creation | yes, multiple, up to the original total |
| meal vouchers | **same month** as payment creation | no — full amount only |
| fringe benefits | **same month** as payment creation | yes |

Outside the window you get **HTTP 400, code 131** — "Payment too old to be refunded". There is no way to
extend it and no test clock in the sandbox, so this path cannot be rehearsed in staging.

## 3. Create the refund

`POST /payments` (`create-a-payment`) with:

- `flow`: `REFUND`
- `parent_payment_uid`: the `id` of the payment being refunded
- `amount_unit`: cents to return — the full original amount for meal vouchers, any amount up to the
  remaining balance otherwise
- `currency`: `EUR`
- `external_code`: your internal order id. Satispay recommends it here specifically so refunds reconcile
  in payment reports, and it is shown to the consumer in the app.

Send an `Idempotency-Key`. A double-fired refund is real money.

## 4. Confirm

The response is a payment object. Read its `status`, and read the parent with
`get-the-details-of-a-payment` to confirm the remaining refundable balance before issuing another
partial.

## Errors

- **131 / 400** — outside the refund window for that instrument.
- **36 / 400** — malformed flow or payment; usually a missing `parent_payment_uid` on a `REFUND` flow.
- **44 / 403** — illegal state transition; the payment is not in a state that can be refunded.

## When you do not know whether the original succeeded

Do not create a refund speculatively. Use `PUT /payments/{id}` with `action: CANCEL_OR_REFUND` — it
cancels if still pending and refunds if already accepted, and tells you which happened.
