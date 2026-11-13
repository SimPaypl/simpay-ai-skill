---
name: simpay-api-integration
description: 'Integrate a merchant/partner system with the SimPay.pl payment gateway (api.simpay.pl) — creating payments (BLIK, bank transfers, PayPal, PayPo/Twisto/BLIK Pay Later, DirectBilling/DCB, SMS Premium), handling IPN webhooks, verifying signatures, checking statuses, and issuing refunds. Use whenever the task involves calling SimPay, receiving SimPay notifications, or debugging a SimPay payment flow.'
---

# SimPay.pl API Integration

Reference for integrating with the SimPay.pl payment gateway REST API. Agent-agnostic:
every example is plain HTTP/JSON or PHP, usable regardless of which coding assistant or
language you're working in.

Full docs index (all pages, PL + EN): https://docs.simpay.pl/llms.txt
Official SDKs: PHP, Laravel, Java, TypeScript, Go, Kotlin, Python — see [SDKs](#sdks) below.

## Before you start

You need, from the SimPay Client Panel:
- A `serviceId` (created per service/shop during registration, e.g. `e65c7519`)
- An API key (**Account → API Keys** — can be scoped, e.g. payments-only / no refunds, and IP-restricted)
- A private **IPN key** (separate from the API key — used only for webhook signature verification)
- An IPN notification URL configured for the service

Keep API and IPN keys on the backend, outside source control, browser bundles, URLs,
and logs. Use separate configuration for each service and environment. Select the
expected service's IPN key from trusted local configuration, not from arbitrary
request-provided configuration. Redact customer data in diagnostic payloads.

Base URL: `https://api.simpay.pl`

Every request:
```
Authorization: Bearer <API_KEY>
Content-Type: application/json
Accept: application/json
```

Errors are standard HTTP status codes with a JSON body; validation errors (422) include
an `errors` object keyed by field. Error responses for a valid API key also include
`feedbackUrl` (see [Reporting API problems](#reporting-api-problems)).

Full OpenAPI/Swagger spec: https://docs.simpay.pl/_bundle/@l10n/en/apis/index.yaml
(treat it as source of truth for exhaustive request/response schemas and error codes —
the prose docs below cover the common path only).

## Creating a payment

```
POST /payment/{serviceId}/transactions
```

```json
{
  "amount": 100.00,
  "currency": "PLN",
  "directChannel": "blik",
  "control": "Order no. 12345",
  "customer": {
    "email": "jan.kowalski@example.com",
    "countryCode": "PL",
    "ip": "203.0.113.7"
  },
  "antifraud": {
    "useragent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
  }
}
```

Response (200): a transaction UUID and `redirectUrl` — redirect the buyer there to complete
payment.

Persist the returned transaction ID with the local order, service ID, expected amount,
currency, and environment before redirecting. A browser redirect, including a success
URL, is never proof of payment. Fulfill only after verified server-side confirmation.
Use decimal arithmetic for money and preserve amount strings during signature checks.

**Two things that silently trigger fraud rejection if wrong:**
- `customer.ip` must be the **buyer's** real IP (from the incoming HTTP request), never your
  server's static/outbound IP.
- `antifraud.useragent` must be the buyer's actual browser `User-Agent` string, forwarded as-is.

When behind a reverse proxy, configure the framework's trusted proxies explicitly.
Only accept forwarded client-IP headers from those proxies. Do not blindly trust a
public request's `X-Forwarded-For`, for either buyer IPs or IPN source-IP validation.

`directChannel` selects the payment method. Common values — full list at
https://docs.simpay.pl/en/payment/methods.md:

| Group | Values |
|---|---|
| BLIK | `blik`, `blik-level0`, `blik-paylater`, `blik-recurrent` |
| Deferred | `paypo`, `twisto`, `twisto-pay-in-3` |
| PL bank transfer | `mbank`, `pko_bp`, `ing`, `pekao_sa`, `santander`, `alior`, `millennium`, `bnp`, `pocztowy`, `citi`, `agricole`, `nest`, `plus`, `velobank`, `bos`, `inteligo`, `spoldzielcze` |
| Other | `paypal`, `paysafecard`, `skrill` |
| LATAM | `latam-pix`, `latam-boleto`, `latam-oxxo`, `latam-spei`, `latam-pse`, `latam-yape`, … (see methods page) |

## Statuses

Transaction: `transaction_new` → `transaction_generated` → `transaction_confirmed` →
`transaction_paid` (terminal success), or `transaction_failure` / `transaction_expired` /
`transaction_fraud_possibility` / `transaction_fraud` (terminal failure) / `transaction_refunded`.

**Only treat `transaction_paid` as paid.** `transaction_confirmed` is not a guarantee of funds.

Refund: `refund_new` → `refund_pending` → `refund_completed` / `refund_rejected` / `refund_failed`.

Subscription (BLIK recurrent): `subscription_pending` → `subscription_active` →
`subscription_cancelled` / `subscription_expired` / `subscription_finished` /
`subscription_fraudulent`.

## Refunds

```
POST /payment/{serviceId}/transactions/{transactionId}/refunds
```

Body `{}` (or omit `amount`) = full refund; `{"amount": "25.50"}` = partial. Response includes
`refundId`, initial status `refund_new`. Constraints: transaction must be ≤30 days old,
production (non-test) only, no duplicate pending refund per transaction, cumulative refunds
can't exceed the original amount.

The transaction must have status `transaction_paid`, and the merchant's SimPay balance
must cover the refund. Save `refundId` alongside the transaction ID and local refund
request. Creating a refund only queues it; mark it completed only on verified
`refund_completed`, not on a successful creation response or `refund_pending`.
Track partial refunds separately. `transaction_refunded` means a completed full refund,
so do not wait for that transaction status to record a completed partial refund.
See https://docs.simpay.pl/en/payment/refunds.md.

## Status checks and recovery after timeouts

Use these authenticated GET requests to reconcile Online Payments operations:

```http
GET /payment/{serviceId}/transactions/{transactionId}
GET /payment/{serviceId}/transactions
GET /payment/{serviceId}/transactions/{transactionId}/refunds/{refundId}
GET /payment/{serviceId}/transactions/{transactionId}/refunds
```

Use IPNs as the primary status source and bounded polling as a fallback. A timeout
after a POST leaves the outcome unknown; it does not prove that creation failed.
Persist the intended operation locally before sending it and prevent concurrent
creation attempts for the same local payment or refund request.

- If you have the transaction or refund ID, retrieve its current state before retrying.
- If a refund creation response was lost, inspect the transaction's refunds before
  issuing another refund. Do not match on amount alone when multiple refunds are possible.
- If payment creation returned no ID, reconcile the service's transaction list and
  verified IPNs with the stored order association. Read the current OpenAPI for list
  pagination and filters; do not invent a payment `filter[control]` parameter.
- `control` is correlation metadata, not a documented idempotency key. Reusing it
  does not guarantee that a repeated POST cannot create another transaction.
- An empty list or delayed IPN alone does not prove failure. If the original operation
  cannot be identified reliably, keep it unresolved for reconciliation rather than
  retrying blindly. A refund that completed before a retry could allow a second refund.

## Online Payments IPN webhooks

This section describes Online Payments IPN v2 only. DirectBilling uses a different
payload and signature contract; see [DirectBilling IPNs](#directbilling-ipns).
SimPay POSTs JSON to the configured IPN URL on status changes.

For a durably accepted Online Payments notification, return either:
- HTTP 200, `Content-Type: text/plain`, body exactly `OK`.
- HTTP 202 or HTTP 204 with an empty body.

Do not assume that any arbitrary 2xx response body is accepted. See
https://docs.simpay.pl/en/notifications/payment/structure.md.
Failed deliveries are retried: every 1 min (×11), then every 5 min (×5), every 30 min
(×5), every 1 hour (×9), then marked permanently undelivered.

### Durable acceptance and duplicate delivery

1. Validate the JSON shape, source IP according to your validation policy, and signature.
2. Check the event type, expected `data.service_id` where supplied, and the transaction's
   association with your local order and service. Validate amount, currency, and environment
   before any fulfillment. Do not use `control` alone to authorize an order update.
3. Atomically store the verified event keyed by `notification_id` in a durable inbox,
   or durably enqueue it with equivalent deduplication. Only then acknowledge receipt.
   Return a failure if durable storage fails so SimPay can retry.
4. Process the persisted work asynchronously. Protect order fulfillment with an atomic
   business-level guard as well: different notifications or concurrent workers must not
   deliver the same order twice. Use a durable outbox or an idempotent downstream operation
   for external side effects that cannot join the database transaction.

A duplicate event already durably accepted can receive the normal success response.
Deduplicating events does not replace business-level idempotency. Do not deduplicate all
events by transaction ID alone, since later refund events must still be processed.
Reconcile conflicting or stale status updates rather than overwriting state in arrival
order. Keep retryable worker failures in durable storage after acknowledging the IPN.

Payload envelope:
```json
{
  "type": "transaction:status_changed",
  "notification_id": "0196bf93-aca4-7253-8a2d-0941a037f25b",
  "date": "2025-05-20T16:36:07Z",
  "data": { "...": "event-specific, see below" },
  "signature": "sha256 hex digest"
}
```

`data` for `transaction:status_changed`:
```json
{
  "id": "UUID",
  "payer_transaction_id": "string",
  "service_id": "string",
  "status": "transaction_paid",
  "amount": {
    "final_currency": "PLN", "final_value": "100.00",
    "original_currency": "PLN", "original_value": "100.00",
    "commission_system": "...", "commission_partner": "...", "commission_currency": "PLN"
  },
  "control": "Order no. 12345",
  "payment": { "channel": "blik", "type": "..." },
  "customer": { "country_code": "PL" },
  "created_at": "2025-05-20T16:36:00Z"
}
```

Other event types (same envelope, different `data`): `subscription:status_changed`,
`blik:alias_status_changed`, `transaction_blik_level0:code_status_changed`,
`transaction_refund:status_changed`, `ipn:test` (send this to test your endpoint before going
live).

**Before fulfilling an order**, check both `status === "transaction_paid"` *and* that
`amount.original_value` / `amount.original_currency` match what you charged — don't trust
`control` alone as the source of truth for the amount.

Also check that `data.id` belongs to that order under the expected `data.service_id`.
An `ipn:test` event must never trigger real fulfillment.

### Signature verification

Not HMAC — **plain SHA-256** over a `|`-joined string, with your private IPN key appended
last:

```
type|notification_id|date|<data values in original JSON key order, flattened>|<your IPN key>
```

Hash with SHA-256, lowercase hex, compare to the `signature` field in the body (the signature
is in the JSON payload, not a header).

Before calling the example below, reject malformed JSON, missing or wrongly typed
envelope fields, a non-object `data`, and signatures that are not 64 lowercase hex
characters. This hash helper assumes an already validated envelope.

```php
function verifySimPaySignature(array $payload, string $ipnKey): bool
{
    $parts = [$payload['type'], $payload['notification_id'], $payload['date']];

    array_walk_recursive($payload['data'], function ($value) use (&$parts) {
        $parts[] = $value;
    });

    $parts[] = $ipnKey;

    $expected = hash('sha256', implode('|', $parts));

    return hash_equals($expected, $payload['signature']);
}
```

Reject invalid signatures without changing business state, for example with HTTP 403.
Do not acknowledge an unverifiable event as successfully processed; a legitimate
delivery with a misconfigured key may need retrying after configuration is corrected.

### Porting signature verification to other languages

The PHP example's conversions are part of the signature contract, not permission to
normalize the payload. Follow these rules before hashing:

- Keep the envelope prefix `type`, `notification_id`, `date` in that order. Recursively
  visit all scalar values in `data` in received object-key and array-element order.
  Do not sort keys, or parse into a model that drops unknown fields before verification.
- Skip absent optional fields completely, without inserting a placeholder or separator.
  For an explicit JSON `null`, append an empty string, keeping its position.
- Include new fields received inside `data`, including nested fields. Exclude only the
  top-level `signature`, not a nested field merely because it has the same name.
- Preserve strings exactly. `"100.00"` must not become `"100"`; do not normalize dates,
  URLs, whitespace, or Unicode, or apply locale-specific number formatting.
- Where numeric or boolean values occur, match the sender's PHP string conversion.
  Python `str(None)` and `str(False)` are not compatible; null becomes `""`, and PHP
  booleans become `"1"` or `""`. Test numeric conversions against sender fixtures rather
  than assuming every language formats floats identically.
- Append the IPN key last, join using `|`, hash the UTF-8 bytes with SHA-256, and compare
  in constant time. For byte-comparison APIs, validate the hex format and length first.

Use test fixtures with null versus absent fields, a decimal string with trailing zeros,
a newly added nested field, changed key order, a tampered amount, and a wrong key.
The null and absent cases must produce different joined strings. Verify the fixtures
against a known sender-generated signature, not an expectation generated by the same
implementation under test. See https://docs.simpay.pl/en/notifications/payment/signature.md.

### IP validation (optional, extra layer)

There is **no static IP allowlist** to hardcode. SimPay's sending IPs can change, so query
fresh on every incoming notification instead of caching:

```
GET https://api.simpay.pl/ip → {"data": ["X.X.X.X", "Y.Y.Y.Y"]}
```

Cross-check the request's source IP against this list. Signature verification is the
primary defense; this is a belt-and-suspenders check, not a replacement for it.

## DirectBilling IPNs

DCB does not use the Online Payments `type` / `notification_id` / `data` envelope.
Its payload contains `id`, `serviceId`, `status`, `values`, `returns`, optional `control`,
`number_from`, `provider`, and `signature`. Route it to a separate parser and verifier;
do not pass it to `verifySimPaySignature` above.

The DCB signature is lowercase SHA-256 of the following values joined with `|`, using
the DirectBilling service's IPN key at the end:

```text
id|serviceId|status|values.net|values.gross|values.partner|returns.complete|returns.failure|control if present|number_from|provider|DCB_IPN_KEY
```

Use the documented field order. If `control` is absent, omit its position entirely;
the text `control if present` above is a placeholder, not a literal signed value.
Preserve value representations and compare signatures in constant time.

Check `serviceId`, `id`, and the expected order association and amount before business
updates. Only `transaction_db_payed` authorizes fulfillment; `transaction_db_confirmed`
does not. Apply the same durable-acceptance and atomic fulfillment rules as above, but
do not expect `notification_id`. Use the service, transaction, and business transition
to prevent duplicate effects while allowing later status updates.

DCB requires HTTP 200 with plain-text body exactly `OK` after durable acceptance.
Do not use Online Payments' empty 202/204 alternatives for DCB.
Contract: https://docs.simpay.pl/en/notifications/directbilling.md.

## Secondary flows (brief — see linked docs before implementing)

- **BLIK Level 0** (`blik-level0`): `POST /payment/{serviceId}/blik/level0/{transactionId}`
  with `ticket.T6` (the 6-digit code). Success is HTTP 204, but that's not proof of payment —
  wait for the `transaction:status_changed` and `transaction_blik_level0:code_status_changed`
  IPNs. PLN only. https://docs.simpay.pl/en/payment/blik/level0.md
- **BLIK Level 0 One-Click** (saved alias, no code on repeat purchases):
  https://docs.simpay.pl/en/payment/blik/level0_oneclick.md
- **BLIK Recurrent** (`blik-recurrent`, subscriptions): register with `amount: 0`, wait for
  `subscription:status_changed` → `subscription_active`, then charge via `subscriptionId`.
  https://docs.simpay.pl/en/payment/blik/recurrent.md
- **DirectBilling / DCB** (charge to phone bill — Orange/Play/Plus/T-Mobile/Netia):
  `POST /directbilling/{serviceId}/transactions`. Only fulfill on `transaction_db_payed`,
  not `transaction_db_confirmed` (authorization only). https://docs.simpay.pl/en/directbilling.md
- **PayPo / Twisto / BLIK Pay Later** (deferred payment / BNPL, PLN only, amount limits apply
  per provider): use the matching `directChannel` value above.
  https://docs.simpay.pl/en/payment/postponed/paypo.md ·
  https://docs.simpay.pl/en/payment/postponed/twisto.md ·
  https://docs.simpay.pl/en/payment/postponed/blik-paylater.md
- **SMS Premium**: prefix-based keyword billing (`SIM`, `CSGO`, `MT2`, `MCRAFT`, `GET`,
  adult-content `SIMP`), price tiers 0.50–25.00 PLN net. https://docs.simpay.pl/en/sms.md

## SDKs

| Language | Repo |
|---|---|
| PHP | https://github.com/SimPaypl/SimPay-API-php |
| Laravel | https://github.com/SimPaypl/simpay-laravel |
| TypeScript | https://github.com/SimPaypl/simpay-typescript |
| Java | https://github.com/SimPaypl/SimPay-API-java |
| Kotlin | https://github.com/SimPaypl/SimPay-API-Kotlin |
| Go | https://github.com/SimPaypl/SimPay-API-go |
| Python | https://github.com/SimPaypl/SimPay-API-Python |

Check whether one exists for your stack before hand-rolling an HTTP client.

## Reporting API problems

When the SimPay API itself gets in your way, tell the SimPay team with
`POST https://api.simpay.pl/feedback`. Any enabled API key works; no extra ability is needed.

Send a report when an endpoint returns a server error or contradicts these docs, when a
response is ambiguous or misleading, or when a capability your integration needs is missing,
even if a retry recovered. Do not report validation errors caused by your own input that you
already fixed.

```http
POST /feedback
Authorization: Bearer <API_KEY>
Content-Type: application/json

{
  "kind": "confusing_result",
  "content": "GET /payment/{serviceId}/transactions returned an empty list without saying whether filters matched nothing. Retry returned the same.",
  "endpoint": "GET /payment/{serviceId}/transactions",
  "correlation_id": "478a94266b77f07125c935c97534b6d2"
}
```

- `kind`: `bug`, `feature_request`, or `confusing_result`.
- `content` (max 4000 chars): expected behavior, actual result, and whether a retry helped.
- `endpoint` and `correlation_id` are optional; copy `correlation_id` from the
  `X-Correlation-ID` response header so SimPay can find the request.
- Remove API keys, IPN keys, customer data, and payment data before sending.
- The API answers `201` with the report `id`, or `429 FEEDBACK_LIMIT_EXCEEDED` after 20
  reports per client in an hour.

## Checklist before going live

- [ ] `customer.ip` is the real buyer IP, not the server's
- [ ] `antifraud.useragent` is forwarded from the buyer's request
- [ ] Forwarded IP headers are trusted only from configured reverse proxies
- [ ] API and IPN secrets remain backend-only and are redacted from logs
- [ ] Verified IPNs are durably stored before acknowledgement; storage failure returns failure
- [ ] Duplicate `notification_id` delivery and concurrent workers cannot duplicate fulfillment
- [ ] Online Payments and DCB use separate verifiers and the correct acknowledgement contract
- [ ] IPN signature verified with the **IPN key**, not the API key (different secrets)
- [ ] Cross-language signature fixtures cover null, absent fields, ordering, and new nested fields
- [ ] Order fulfillment gated on `status === "transaction_paid"` + amount match, not just
      "an IPN arrived"
- [ ] Service and transaction IDs match the stored order; browser redirects never mark it paid
- [ ] Timed-out creation is reconciled before retry; `control` is not treated as idempotency
- [ ] `ipn:test` notification received and handled correctly before switching off test mode
- [ ] Refund flow tested (only works on production transactions, ≤30 days old)
- [ ] Refund eligibility includes paid status and sufficient balance; `refundId` is persisted
- [ ] Only verified `refund_completed` marks a refund completed
