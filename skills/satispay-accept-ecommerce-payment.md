---
name: satispay-accept-ecommerce-payment
description: >-
  Take a one-off Satispay payment on an e-commerce checkout using the MATCH_CODE (QR) flow, and settle
  the outcome correctly via the server-to-server callback rather than the browser redirect.
api: Satispay GBusiness API
base_url: https://authservices.satispay.com/g_business/v1
operations:
- create-a-payment
- get-the-details-of-a-payment
- update-a-payment
generated: '2026-08-26'
method: generated
source: >-
  openapi/satispay-gbusiness-api.json, https://developers.satispay.com/docs/web-redirect-pay,
  https://developers.satispay.com/reference/callback-s2s
---

# Accept a one-off Satispay payment

## Before you start

You need a `KeyId` paired to the shop and an RSA private key. Every request is signed — see
`satispay-authenticate-and-sign.md`. Amounts are integer cents: `€12.40` is `1240`. `EUR` is the only
currency.

## 1. Create the payment

`POST /payments` (`create-a-payment`).

- `flow`: `MATCH_CODE` for a QR the consumer scans, `MATCH_USER` when you already hold a `consumer_uid`.
- `amount_unit`: cents, integer.
- `currency`: `EUR`.
- `external_code`: your own order id. Set it. It lands in the merchant Dashboard export and is shown to
  the consumer in the app, and it is what makes reconciliation possible later.
- `callback_url`: required in practice. Use the `{uuid}` placeholder, e.g.
  `https://example.com/satispay-callback?payment_id={uuid}`.
- `redirect_url`: where the browser returns after the app hands back.
- `expiration_date`: ISO 8601, optional.

Send `Idempotency-Key` on this POST. If the call times out, retry with the **same** key — Satispay
returns the original response rather than creating a second payment.

The response carries `id`, `code_identifier` (the Dynamic Code you render as the QR), and `status`.

## 2. Wait for the callback — do not trust the redirect

Satispay's own documentation warns that a consumer can close the browser after scanning and still
confirm in the app. If you create the order from the redirect, that payment is taken and no order
exists.

The callback is a bare `GET` to your `callback_url` with **no body and no status**. It tells you only
that something changed.

- Every callback carries a `Digest` header (SHA-256 of the body) you can use for integrity.
- An `Authorization` signature header is *conditional*. Satispay states not all callbacks are signed;
  whether yours are depends on how the integration was configured. Do not build a receiver that rejects
  unsigned callbacks unless you have confirmed with Satispay that signing is on for your integration.
- Satispay retries a non-2XX response 3 times at 1s, 2s and 4s, then stops. Return 2XX fast and do the
  work asynchronously.
- Throttle your own endpoint. Concurrent callback volume scales with concurrent payments.

## 3. Read the real status

Read `payment_id` from the query string and call `GET /payments/{id}`
(`get-the-details-of-a-payment`). `status` is what matters:

- `ACCEPTED` — money is taken. Create the order.
- `PENDING` — not settled yet. Do nothing irreversible.
- `CANCELED` — do not fulfil.

Also poll after the retry window closes. Satispay explicitly recommends periodic status checks as the
backstop when callbacks fail.

## 4. Cancel or capture

`PUT /payments/{id}` (`update-a-payment`) with `action`:

- `ACCEPT` — capture a `PENDING` or `AUTHORIZED` payment. `amount_unit` may be lower than what was
  locked.
- `CANCEL` — cancel a `PENDING` or `AUTHORIZED` payment.
- `CANCEL_OR_REFUND` — **use this when you do not know the state.** It cancels a `PENDING` or
  `AUTHORIZED` payment, or refunds one already `ACCEPTED`. Satispay recommends it precisely for the
  timeout / HTTP 500 case. Read the returned status to learn which outcome you got.

## Errors worth handling by name

| code | HTTP | meaning | what to do |
|---|---|---|---|
| 21 | 400 | Insufficient availability | Consumer cannot fund it. Offer another method. |
| 27 | 400 | Payment not allowed | May be temporary. Retry later, do not loop. |
| 44 | 403 | Illegal state transition | Read the status first; use `CANCEL_OR_REFUND`. |
| 70 | 403 | Anti-hammering violation | This is the rate limit. Back off exponentially — there is no `Retry-After`. |
| 41 | 404 | Resource not found | Often a staging KeyId against production. Check the base URL. |

Errors are `{"code","message","wlt"}` — not RFC 9457. Keep `wlt`; it is what Satispay support traces on.
Retry every non-2xx, always carrying the original `Idempotency-Key`.
