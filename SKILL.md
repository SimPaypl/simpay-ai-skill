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

Base URL: `https://api.simpay.pl`

Every request:
```
Authorization: Bearer <API_KEY>
Content-Type: application/json
Accept: application/json
```

Errors are standard HTTP status codes with a JSON body; validation errors (422) include
an `errors` object keyed by field.

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

**Two things that silently trigger fraud rejection if wrong:**
- `customer.ip` must be the **buyer's** real IP (from the incoming HTTP request), never your
  server's static/outbound IP.
- `antifraud.useragent` must be the buyer's actual browser `User-Agent` string, forwarded as-is.

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

## IPN webhooks (payment notifications)

SimPay POSTs a JSON body to your configured IPN URL on every status change. Respond
**HTTP 200 with body `OK`** (plain text) — quickly. Do slow work (fulfillment, emails)
asynchronously after acknowledging, not inline in the handler. Non-2xx responses trigger
retries: every 1 min (×11), then every 5 min (×5), every 30 min (×5), every 1 hour (×9),
then the notification is marked permanently undelivered — so a slow handler that times out
looks identical to a broken one from SimPay's side.

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

### Signature verification

Not HMAC — **plain SHA-256** over a `|`-joined string, with your private IPN key appended
last:

```
type|notification_id|date|<data values in original JSON key order, flattened>|<your IPN key>
```

Hash with SHA-256, lowercase hex, compare to the `signature` field in the body (the signature
is in the JSON payload, not a header).

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

Reject the notification (but still return 200 so SimPay doesn't retry a forged request
forever — log and drop it) if the signature doesn't match.

### IP validation (optional, extra layer)

There is **no static IP allowlist** to hardcode. SimPay's sending IPs can change, so query
fresh on every incoming notification instead of caching:

```
GET https://api.simpay.pl/ip → {"data": ["X.X.X.X", "Y.Y.Y.Y"]}
```

Cross-check the request's source IP against this list. Signature verification is the
primary defense; this is a belt-and-suspenders check, not a replacement for it.

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

## Checklist before going live

- [ ] `customer.ip` is the real buyer IP, not the server's
- [ ] `antifraud.useragent` is forwarded from the buyer's request
- [ ] IPN handler responds `200 OK` fast and does fulfillment work async
- [ ] IPN signature verified with the **IPN key**, not the API key (different secrets)
- [ ] Order fulfillment gated on `status === "transaction_paid"` + amount match, not just
      "an IPN arrived"
- [ ] `ipn:test` notification received and handled correctly before switching off test mode
- [ ] Refund flow tested (only works on production transactions, ≤30 days old)
