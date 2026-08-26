---
name: satispay-reporting-and-reconciliation
description: >-
  Pull Satispay transaction data for reconciliation — list payments, generate and retrieve reports, and
  read the daily closure — and match it back to your own order ids.
api: Satispay GBusiness API
base_url: https://authservices.satispay.com/g_business/v1
operations:
- get-list-of-payments
- create-new-report
- get-list-of-reports
- retrieve-a-report
- retrieve-daily-closure
- retrieve-shop-profile
generated: '2026-08-26'
method: generated
source: >-
  openapi/satispay-gbusiness-api.json, openapi/satispay-production.json,
  https://developers.satispay.com/reference/external-code,
  https://developers.satispay.com/changelog/changelog-187
---

# Reconcile Satispay transactions

## The join key

`external_code` is the field that makes reconciliation work. Set it to your order id on **every**
`create-a-payment`, refunds included. It appears in the Dashboard payments export and in the API, and it
is shown to the consumer in the app. If you did not set it, you are matching on amount and timestamp.

## List payments

`GET /payments` (`get-list-of-payments`). Cursor pagination:

- `limit` — page size. No default or maximum is published; start conservative.
- `starting_after` — the `id` of the last payment from the previous page.
- `starting_after_timestamp` — milliseconds, to start from a point in time.
- `status` — filter to `ACCEPTED`, `PENDING` or `CANCELED`.

The envelope is `{ has_more, data }`. Loop while `has_more` is true.

Remember `amount_unit` is **cents as an integer**: `150` is `€1.50`. Do not divide twice.

## Generate a report

For anything larger than a page-through, use reports.

1. `POST /reports` (`create-new-report`). As of documentation version 1.8.7 the columns are
   configurable, so ask for the ones you need rather than post-processing a fixed shape. Meal Voucher
   and Fringe Benefit columns arrived in 1.8.0; Buy Now Pay Later activation columns in 1.8.1.
2. `GET /reports` (`get-list-of-reports`) to see what exists.
3. `GET /reports/{report_id}` (`retrieve-a-report`) to fetch one.

Report errors are specific: **189** invalid payload, **190** invalid query type, **244** invalid report
merchant id — all HTTP 400.

Reports are generated asynchronously and cannot be deleted through the API.

## Daily closure

`GET /daily_closure/{daily_closure_date}` (`retrieve-daily-closure`) — addressed by date, not by id.
This is the end-of-day figure to settle your books against.

## Shop context

`GET /profile/me` (`retrieve-shop-profile`) returns the shop behind the KeyId. Note that tenancy is
carried entirely by the credential — no payment operation takes a shop id — so **one KeyId is one shop**.
If you run multiple shops you hold multiple KeyIds, and reconciliation runs per KeyId.

## Refunds in the ledger

A refund is not a separate object type. It is a payment with `flow: REFUND` whose `parent_payment_uid`
points at what it reversed. When you total a period, treat REFUND-flow payments as negatives against
their parent rather than as independent transactions.

## Dates

`yyyy-MM-dd'T'HH:mm:ss.SSSZ`, e.g. `2016-02-11T11:20:49.000Z`.
