# Webhooks

Live pages: https://docs.fianto.xyz/developers/webhooks/overview.md,
https://docs.fianto.xyz/developers/webhooks/verify.md,
https://docs.fianto.xyz/developers/webhooks/events.md (every payload in full),
https://docs.fianto.xyz/developers/webhooks/retries-and-delivery.md

## Endpoint and the verification challenge

One webhook endpoint per application, set in the dashboard (Developers → application → Manage
webhook). URL rules: `https`; no user/password/`#fragment`; port 443 or ≥1024; public IP (re-checked
every attempt); ≤2,048 chars.

A new URL is **pending** until it passes a signed `endpoint.verification` request:

```json
{ "type": "endpoint.verification", "timestamp": "2026-09-28T10:00:00.000Z", "data": { "challenge": "…" } }
```

Answer within **5 s** with HTTP **200** and `{"challenge":"<the value received>"}`. Other statuses fail;
redirects are not followed. Checks: 10/hour per endpoint, 30/hour per account (`too_many_probes`).
The SDK handlers answer the challenge themselves — once they have the right secret. If the first check
failed because the secret was missing, fix it and click "Verify again".

**Events published before the application's URL is first verified are stored but never
delivered**, even after you verify one later (read them with `GET /v1/events`, kept 90 days). Verify
the URL before taking payments. (A URL that was verified and later suspended *holds* its events
instead — see Delivery semantics.)

The signing secret (`whsec_…`) appears via "Reveal signing secret" once the application has a URL.

## Signatures (Standard Webhooks)

| Header | Value |
|---|---|
| `webhook-id` | The event id `evt_…`; same on every retry/redelivery |
| `webhook-timestamp` | Unix seconds of **this attempt** (each retry is re-signed) |
| `webhook-signature` | `v1,<base64 HMAC-SHA256 of "id.timestamp.body">`, key = base64-decoded part of the secret after `whsec_`. Space-separated when several (two during a secret roll) |

Any Standard Webhooks library works, but prefer the SDK. Timestamp tolerance is the receiver's job:
the SDK rejects >300 s skew by default (settable 1–3,600, never off). Keep the server clock synced.

## Envelope and event types

```json
{ "id": "evt_<32 hex>", "type": "order.paid", "timestamp": "…", "data": { … } }
```

`GET /v1/events[/{id}]` returns the same envelope plus `object: "event"`.

| Event | When |
|---|---|
| `checkout.session.completed` | Session's payment finalized while still `OPEN` (both modes) |
| `checkout.session.expired` | Nobody paid before `expires_at` (sweep, or replaced by a new create) |
| `checkout.session.canceled` | You or the payer cancelled; also when a subscription session's link went out of date. No order event |
| `order.paid` | Order's payment finalized → **fulfil here**. Also for a late payment on an expired/cancelled session (unless a newer session exists) |
| `order.expired` | With `checkout.session.expired` from the sweep. An `EXPIRED` order can still become `PAID` |
| `order.duplicate_payment` | Another transfer with the session's reference arrived while the order was `PAID` or had another live payment. `data.duplicate` = `{ signature, amount }`. The USDC is in the merchant's wallet |
| `subscription.created` | Subscribe transaction finalized; `ACTIVE`, first period paid → grant access |
| `subscription.renewed` | A renewal finalized; `PAST_DUE` → `ACTIVE` |
| `subscription.payment_failed` | Every counted renewal failure; `data.failure = { reason, attempt_no, next_attempt_at }` |
| `subscription.past_due` | First counted failure of an `ACTIVE` subscription (sent once, with `payment_failed`) |
| `subscription.cancel_scheduled` | Period-end cancel scheduled (`cancel_reason` `MERCHANT_CANCELED` / `PAYER_CANCELED`) |
| `subscription.cancel_withdrawn` | Payer resumed; renewals continue |
| `subscription.ended` | `ENDED`; `end_reason` `MERCHANT_CANCELED` / `PAYER_CANCELED` / `PAYMENT_FAILED` |
| `test.event` | You asked for it (API, CLI, dashboard). `data` has only a message |

That is 14 types (+ `endpoint.verification`). There is **no** `payment.succeeded`,
`payment.completed`, `charge.*` or `invoice.*`. Answer 2xx to types you don't recognise — more may be
added.

Payload rules:

- `data.order_id` is **your** id (top level of `data`, not in `metadata`). fianto ids are prefixed:
  `fian_cs_` session, `fian_ord_` order, `fian_pay_` payment, `fian_cus_` customer, `fian_sub_`
  subscription, `fian_plan_` plan, `fian_price_` price.
- Amounts in payloads (`amount`, `fee_amount`, `total_amount`, `period.amount_due`, `period.fee_due`,
  `duplicate.amount`) are **strings of USDC base units, 6 decimals**: `"25000000"` = 25 USDC. Compare
  against your price with `usdc.toBaseUnits('25.00')` or format with `usdc.format(...)` — never floats.
  `amount` = your price (what you receive), `fee_amount` = service fee, `total_amount` = what the payer paid.
- Statuses are UPPERCASE; `mode` is lowercase. Absent values are `null`, not missing keys.
- `customer.id` is `null` until a payment settles; on an expired order `customer.id` and
  `customer.wallet` are `null`.
- Order objects (`order.*`), main fields: `object`, `id`, `order_id`, `checkout_session_id`, `status`, `amount`, `fee_amount`,
  `total_amount`, `currency`, `customer { id, wallet, email, reference }`, `payment { id, signature }`,
  `metadata`, `created_at`, `paid_at`, `expired_at`.
- Subscription objects (`subscription.*`), main fields: `object`, `id`, `status`, `order_id`, `price_id`, `plan_id`,
  `product_name`, amounts, `interval` (`MONTH` = 30 days / `YEAR` = 365 days), `period_hours`,
  `customer`, `current_period_start/end`, `cancel_at_period_end`, `cancel_at`, `cancel_reason`,
  `end_reason`, `payment`, `metadata`, plus `period` or `failure` on the events that carry them.

## Delivery semantics

- **At least once**: record each `event.id` in the **same database transaction** as the work it
  triggers; skip ids already seen.
- **Unordered**: when current state matters, read the order/subscription back from the API.
- **Fast**: any 2xx within **10 s** = delivered. Do slow work after answering, or in a queue.
- Each event gets up to **8 attempts** (7 retries; waits 1 min, 5 min, 30 min, 2 h, 5 h, 10 h, 10 h; ±20 %;
  ~28 h total), then `EXHAUSTED`.
- **5 exhausted deliveries in a row** suspend the endpoint (URL back to pending, reason
  `delivery_failures`). New and waiting deliveries are **held** (up to 10,000); fix the route and click
  "Verify again" → held deliveries are sent. Any success resets the count.
- Redeliver from the dashboard only (no API), same `webhook-id`, 60/hour per application. Delete the id
  from your dedupe table first if you want it processed again.
- Catch up after an outage with `GET /v1/events` (newest first, 90-day retention).

## SDK handler (recommended)

`Webhooks()` (`@fianto/nextjs`), `webhooks()` (`@fianto/express`, `@fianto/hono`), or
`createWebhookHandler()` (`@fianto/sdk/handlers`). It reads the raw body, verifies signature and
timestamp, answers `endpoint.verification`, and calls one callback per type.

| Option | Default / notes |
|---|---|
| `secret` | `FIANTO_WEBHOOK_SECRET` read per request; string or array (during a roll). `[]` throws |
| `toleranceSeconds` | 300 (1–3,600) |
| `maxBodyBytes` | 1 MiB → larger bodies get 413 `{"error":"payload_too_large"}` |
| `onCheckoutSessionCompleted`, `onCheckoutSessionExpired`, `onCheckoutSessionCanceled`, `onOrderPaid`, `onOrderExpired`, `onOrderDuplicatePayment`, `onSubscriptionCreated`, `onSubscriptionRenewed`, `onSubscriptionPaymentFailed`, `onSubscriptionPastDue`, `onSubscriptionCancelScheduled`, `onSubscriptionCancelWithdrawn`, `onSubscriptionEnded`, `onTestEvent` | One per type, typed `event` |
| `onEvent` | After the type's callback, for every verified event except the challenge |
| `onVerificationError(error)` | Why a request was refused (`error.reason`). Wire it: the response never says |
| `onError(error, event)` | A callback threw → response is 500 so fianto retries |

Responses: challenge → `200 {"challenge"}`; handled → `200 {"received":true}`; failed check →
`400 {"error":"invalid_webhook"}`; callback threw → `500 {"error":"handler_failed"}`; non-POST → 405
(Express/Hono `app.post` routes answer 404 instead).

Framework rules:

- **Next.js**: a plain App Router Route Handler, `export const POST = Webhooks({...})`. No middleware may
  read or rewrite the body.
- **Express**: mount **before** `express.json()` / `express.urlencoded()` / any body reader. A global
  parser registered earlier makes the middleware fail with `BodyAlreadyParsedError` (500), and adding
  `express.raw()` to the route does **not** help. Either move fianto routes first or scope the parser
  (`app.use('/api', express.json())`).
- **Hono**: no earlier `app.use(...)` may read the body. On Cloudflare Workers build the handler inside
  the route with `secret: c.env.FIANTO_WEBHOOK_SECRET` (see sdk.md).
- Proxies must pass the body and the three `webhook-*` headers through unchanged.

## Manual verification with `verifyWebhook`

```ts
import { isWebhookVerificationError, verifyWebhook } from '@fianto/sdk/webhooks';

export async function POST(request: Request) {
  const body = await request.arrayBuffer(); // raw bytes, never request.json()
  let event;
  try {
    event = await verifyWebhook(body, request.headers, { secret: process.env.FIANTO_WEBHOOK_SECRET });
  } catch (error) {
    if (isWebhookVerificationError(error)) {
      return Response.json({ error: 'invalid_webhook' }, { status: 400 });
    }
    throw error;
  }
  switch (event.type) {
    case 'endpoint.verification':
      return Response.json({ challenge: event.data.challenge }); // you must answer this yourself
    case 'order.paid':
      // dedupe event.id + fulfil event.data.order_id in one DB transaction
      break;
    default:
      break; // 2xx for everything else, including future types
  }
  return Response.json({ received: true });
}
```

In Express use `express.raw({ type: '*/*' })` on that route (before any global JSON parser) and pass
`req.body` (a Buffer). `rawBody` may be a string, `Uint8Array` or `ArrayBuffer` — never re-serialised
JSON. Failure `reason`s: `missing_headers`, `invalid_signature_header`, `timestamp_out_of_tolerance`,
`invalid_secret`, `no_matching_signature`, `invalid_payload`.

## Rolling the signing secret

Dashboard → "Roll signing secret": the old secret stops "Immediately", "In 1 hour" (default) or "In
24 hours". During the grace period every delivery carries two signatures. Deploy the new secret
before the old one stops; the handler accepts both meanwhile:

```ts
const secrets = [process.env.FIANTO_WEBHOOK_SECRET, process.env.FIANTO_WEBHOOK_SECRET_OLD]
  .filter((s): s is string => Boolean(s));
export const POST = Webhooks({ secret: secrets.length ? secrets : undefined, onOrderPaid: async () => {} });
```

At most two secrets are valid at once.

## Troubleshooting

| Symptom | Cause → fix |
|---|---|
| 400 `invalid_webhook`, reason `no_matching_signature` | Wrong `whsec_` (another application's?) or the body was changed before the route |
| reason `invalid_secret` | `FIANTO_WEBHOOK_SECRET` not loaded in this process (Node needs `--env-file`/`dotenv`; Workers needs `c.env`) |
| reason `timestamp_out_of_tolerance` | Server clock >300 s off → NTP |
| reason `missing_headers` | A proxy dropped `webhook-*` headers |
| Express 500 `BodyAlreadyParsedError` | A body parser ran first → mount fianto routes before it |
| "did not echo the challenge back" | Route answered 200 without `{"challenge": …}` |
| "did not answer with HTTP 200" | Wrong path (404) or wrong secret (400) |
| "answered with a redirect" | Register the exact final URL |
| 500 `handler_failed` | Your callback threw; see `onError`. fianto retries |
