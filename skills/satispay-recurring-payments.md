---
name: satispay-recurring-payments
description: >-
  Obtain a consumer's authorization for recurring Satispay charges, charge against it, and revoke it.
  The authorization is the durable object — losing track of it is how a subscription keeps charging.
api: Satispay GBusiness API
base_url: https://authservices.satispay.com/g_business/v1
operations:
- create-authorization
- get-authorization
- update-authorization
- create-a-payment
- get-the-details-of-a-payment
generated: '2026-08-26'
method: generated
source: >-
  openapi/satispay-production.json, openapi/satispay-gbusiness-api.json,
  https://developers.satispay.com/docs/web-redirect-auth,
  https://developers.satispay.com/reference/callback-s2s
---

# Recurring payments with Satispay

## The model

One long-lived **pre-authorized payment token** (an *authorization*) funds many payments. The consumer
grants it once, in the Satispay app, and can revoke it from the app at any time without telling you.
That last fact drives everything below.

## 1. Create the authorization

`POST /g_business/v1/pre_authorized_payment_tokens` (`create-authorization`):

- `reason` — shown to the consumer. Say what you will charge for.
- `callback_url` — **set this.** It is the only way you learn the consumer revoked it. Use the `{uuid}`
  placeholder: `https://example.com/satispay-callback?authorization_id={uuid}`.
- `redirect_url` — where the browser returns.
- `metadata` — your own keys.

Present it with the Web Button (embedded in your page) or the Web Redirect (Satispay-hosted); on mobile,
hand off to the Satispay app. The consumer scans a QR or types their phone number and confirms.

## 2. Store the token id

The returned `id` is what you charge against. Persist it alongside your subscription record.

## 3. Charge

`POST /payments` (`create-a-payment`):

- `flow`: `PRE_AUTHORIZED` (or `PRE_AUTHORIZED_FUND_LOCK` to lock funds now and capture later)
- `token`: the authorization id
- `amount_unit`, `currency: EUR`
- `external_code`: your invoice or billing-period id
- `callback_url`: the payment callback

> `pre_authorized_payments_token` is the older field name for this and the spec describes it as replaced
> by `token`. Use `token`.

Send an `Idempotency-Key` per billing period, derived from something stable like
`sub_<id>_<period>` — that way a retried billing run cannot charge twice.

## 4. Handle revocation

On an authorization callback, call `GET /pre_authorized_payment_tokens/{id}` (`get-authorization`) and
read `status`. The callback carries no status of its own.

If the authorization is no longer valid, charging it returns **HTTP 400, code 172** —
"Pre authorized payments token is not valid". Treat 172 as *stop billing and re-acquire consent*, not as
a transient error to retry.

## 5. Revoke it yourself

`PUT /g_business/v1/pre_authorized_payment_tokens/{id}` (`update-authorization`) with `status` set to
cancel it. Do this when the subscription ends. An authorization you leave live is a standing claim on
someone's account.

## Reversibility

Individual charges refund like any payment — new payment, `flow: REFUND`, `parent_payment_uid`, within
365 days for e-money. Revoking the authorization stops *future* charges; it does not undo past ones.
