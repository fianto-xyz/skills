# v1 API: auth, errors, idempotency, pagination, rate limits

Live pages: https://docs.fianto.xyz/developers/overview.md,
`/developers/{authentication,errors,idempotency,pagination-and-rate-limits}.md`

## Basics

- Base URL `https://api.fianto.xyz`; every path starts `/v1/`. Bodies are JSON, **snake_case**.
- Applications are created in the dashboard (password + 2FA). Each has an app id `fian_app_…`, an app
  secret `fian_sk_live_…` (shown once; the prefix is the same on every deployment), and one webhook endpoint.
  Limits: 10 active / 100 total applications per merchant.
- Products and prices are merchant-wide and **read-only** in v1. Sessions, orders, payments,
  subscriptions and events are scoped to the calling application.
- Keys work only while the merchant account is approved and active.

## The 20 endpoints

| Area | Endpoints |
|---|---|
| Application | `GET /v1/application` |
| Checkout sessions | `POST /v1/checkout-sessions`, `GET /v1/checkout-sessions/{id}`, `POST /v1/checkout-sessions/{id}/cancel`, `POST /v1/checkout-sessions/{id}/link` |
| Orders | `GET /v1/orders`, `GET /v1/orders/lookup?order_id=`, `GET /v1/orders/{id}` |
| Payments | `GET /v1/payments`, `GET /v1/payments/{id}` |
| Subscriptions | `GET /v1/subscriptions`, `GET /v1/subscriptions/{id}`, `POST /v1/subscriptions/{id}/cancel` |
| Products / prices | `GET /v1/products`, `GET /v1/products/{id}`, `GET /v1/prices`, `GET /v1/prices/{id}` |
| Events | `GET /v1/events`, `GET /v1/events/{id}` |
| Webhooks | `POST /v1/webhook/test-event` |

Nothing else exists: no refunds, no customers API, no product/price writes, no redelivery API,
no webhook-endpoint management API.

## Authentication: HTTP Basic

```text
Authorization: Basic base64(app_id:app_secret)
```

```bash
curl https://api.fianto.xyz/v1/application --user "$FIANTO_APP_ID:$FIANTO_APP_SECRET"
```

No bearer tokens, no `X-App-Id` header, no publishable/browser key, no test keys. Every credential
problem (bad header, unknown/revoked secret, disabled app, merchant not approved/active) → 401
`invalid_api_credentials` with `WWW-Authenticate: Basic realm="fianto"`; the SDK throws
`AuthenticationError`.

Rolling a secret (dashboard): old secret stops "Immediately", "In 1 hour" (default) or "In 24 hours";
both work in between (max two valid). Disabling an application is permanent (revokes secrets, disables
its webhook, cancels pending deliveries).

## Error body

```json
{
  "statusCode": 409,
  "error": "Conflict",
  "code": "order_already_paid",
  "message": "That order_id is already paid. Use a new order_id to charge again.",
  "request_id": "req_…",
  "field": "order_id"
}
```

Branch on `code` (open-ended: treat unknown codes by HTTP status). `details` lists every problem when
there were several. `request_id` is also the `X-Request-Id` header; send your own `X-Request-Id`
(8–64 of `[A-Za-z0-9_-]`) to correlate logs. Quote it to support@fianto.xyz.

Common codes: 400 `validation_failed` (also for unknown fields), 401 `invalid_api_credentials`,
400 `idempotency_key_required` / `idempotency_key_invalid`, 422 `idempotency_key_reused`,
409 `idempotency_request_in_progress`, 409 `idempotency_response_unreadable`, 400
`amount_or_price_required`, 400 `amount_out_of_range`, 400 `url_insecure`, 400
`expires_at_out_of_range`, 404 `price_not_found`, 409 `order_already_paid`, 409 `payment_in_progress`,
409 `session_not_reissuable`, 409 `session_not_cancelable`, 409 `checkout_unavailable`, 422
`merchant_token_account_missing`, 400 `price_id_required` / `amount_not_allowed`, 422
`price_not_recurring`, 429 `plan_limit_reached`, 409 `subscription_already_ended`, 409
`webhook_endpoint_not_active`, 429 `rate_limited`, 500 `internal_error`.

### SDK error classes

All extend `FiantoError`. `APIError` (`status`, `code`, `message`, `field`, `details`, `requestId`,
`headers`) with subclasses `AuthenticationError` 401, `PermissionDeniedError` 403,
`InvalidRequestError` 400/422, `NotFoundError` 404, `ConflictError` 409, `RateLimitError` 429
(`retryAfterSeconds`), `ServiceUnavailableError` 503, `InternalServerError` other 5xx. Plus
`ConnectionError`, `TimeoutError` (both carry `idempotencyKey` after a failed POST), `AbortError`,
`UsdcError`.

Check codes with `isFiantoError(err, ErrorCode.OrderAlreadyPaid)` / `isAPIError(err)` rather than
`instanceof` (dual ESM/CJS loads break `instanceof`).

SDK retries (default `maxRetries` 2): network errors, timeouts, 408, 429, 5xx, and 409 only for
`idempotency_request_in_progress` / `checkout_unavailable`. Honours `Retry-After` (throws at once if
>10 s), else backoff 500 ms doubling to 8 s ±25 %. Every retry of a POST reuses the same key.

## Idempotency

Every v1 POST (create session, cancel session, reissue link, cancel subscription, test event) needs
`Idempotency-Key`: 1–255 printable ASCII, no spaces, per application, kept **24 h**. Same key + same
request → the stored response with `Idempotent-Replayed: true`. A failed first request releases the
key.

This matters most for creating a session, because `url` is only in the create response. **Retry with
the same key.** A new key re-runs the request: the second create finds the open session and returns it
with `url: null` (then you must `reissueLink`).

The SDK adds a `crypto.randomUUID()` key per call, reused across its own retries. If your process may
crash between create and saving `url`, derive the key from your data:

```ts
await fianto.checkoutSessions.create(params, { idempotencyKey: `checkout:${orderId}:${attempt}` });
```

When the SDK gives up on a network failure, reuse the key it used:

```ts
import { ConnectionError, Fianto, TimeoutError } from '@fianto/sdk';
import type { CheckoutSessionCreateParams } from '@fianto/sdk';

const fianto = new Fianto();

export async function createWithRecovery(params: CheckoutSessionCreateParams) {
  try {
    return await fianto.checkoutSessions.create(params);
  } catch (err) {
    if ((err instanceof ConnectionError || err instanceof TimeoutError) && err.idempotencyKey) {
      return fianto.checkoutSessions.create(params, { idempotencyKey: err.idempotencyKey });
    }
    throw err;
  }
}
```

| Error | Fix |
|---|---|
| 400 `idempotency_key_required` | Add the header (SDK does) |
| 400 `idempotency_key_invalid` | 1–255 printable ASCII, no spaces |
| 422 `idempotency_key_reused` | Same key, different request → new key for a new request (not retried) |
| 409 `idempotency_request_in_progress` | Retry with the same key in a few seconds (SDK does) |
| 409 `idempotency_response_unreadable` | Already done; read the resource (e.g. `GET /v1/orders/lookup?order_id=`) before resending under a new key |

## Pagination

Lists answer `{ items, next_cursor }`; `limit` 1–100 (default 20), `cursor` = previous `next_cursor`.
Orders/payments/subscriptions/products/prices use numeric cursors (a full last page still returns a
cursor; stop on an empty page or `null`). Events use `evt_…` cursors and end with `null`. The SDK's
`for await` handles both.

## Rate limits

300 req/min per IP; 120 writes + 600 reads/min per application; 300 writes/min per merchant; 10 test
events/hour per application (shared with the dashboard button). Over → 429 `rate_limited` +
`Retry-After`.
