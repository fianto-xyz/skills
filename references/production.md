# Production: performance and reliability

Live pages: https://docs.fianto.xyz/developers/webhooks/retries-and-delivery.md,
`/developers/pagination-and-rate-limits.md`, `/developers/errors.md`, `/developers/idempotency.md`

The fianto facts behind each recommendation are cited inline; the recommendations themselves are
standard engineering for those facts.

## Webhook route: acknowledge fast, work in the background

- fianto counts any 2xx within **10 s** as delivered; slower is a failed attempt even if the work
  finished. After **5 exhausted deliveries in a row** the endpoint is suspended.
- So: verify → dedupe-insert `event.id` → update the order row → enqueue slow work (emails,
  shipping, ERP calls) → return. Keep the handler to one short database transaction.
- If fulfilment work is queued, make the queue job idempotent by `event.id` / `order_id` too.
- Return 500 (throw) only for failures a retry can fix; fianto retries 7 more times over ~28 h. For
  an event you will never handle, return 2xx.

## Don't poll fianto; read your own database

- Webhooks already push every state change. Your success/"confirming" page should read **your**
  order row (written by the webhook), polling your own endpoint every few seconds while it shows
  "confirming".
- Fall back to one server-side `fianto.orders.retrieveByOrderId(orderId)` only when your row is still
  pending after a while (e.g. the webhook URL is not verified yet). Limits: 600 reads/min per
  application, 300 requests/min per IP.

## Reconcile instead of trusting delivery alone

Deliveries are at least once, but a suspended or never-verified URL, or a bug, can leave gaps.
Run a periodic job (e.g. hourly):

```ts
import { Fianto } from '@fianto/sdk';

const fianto = new Fianto();
const LOOKBACK_MS = 48 * 60 * 60 * 1000; // longer than the ~28 h delivery-retry window

export async function reconcile() {
  const oldest = Date.now() - LOOKBACK_MS;
  // Events come newest first, but deliveries are unordered and retried for ~28 h, so an older
  // event can be missing while newer ones were processed: skip processed ids, never stop at one.
  for await (const event of fianto.events.list({ limit: 100 })) {
    if (Date.parse(event.timestamp) < oldest) break;
    if (await alreadyProcessed(event.id)) continue;
    await handleEvent(event); // the same idempotent code your webhook route calls
  }
}

declare function alreadyProcessed(id: string): Promise<boolean>; // your dedupe table
declare function handleEvent(event: unknown): Promise<void>;
```

Events are kept **90 days**. Webhook `order.paid` and `GET /v1/events` share the same envelope, so
one handler serves both.

## Client and retries

- Create one `Fianto` client per process (module scope) and reuse it; on Cloudflare Workers create
  it per request from `c.env`, since secrets exist only there.
- Defaults: `timeoutMs` 30 000 per attempt, `maxRetries` 2 (network errors, timeouts, 408, 429,
  5xx, and 409 `idempotency_request_in_progress` / `checkout_unavailable`). A `Retry-After` over
  10 s is thrown at once (for a 429, as `RateLimitError` with `retryAfterSeconds`) — back off in your
  job queue rather than blocking a request.
- Checkout creation sits on the payer's critical path: consider a lower per-call `timeoutMs` there
  and rely on the same `idempotencyKey` to recover (see api.md). Never retry a create with a new key.
- Pass `signal` (an `AbortSignal`) to tie a call to the incoming request's lifetime.

## Rate-limit budget

| Limit | Scope |
|---|---|
| 300 requests/min | per IP |
| 120 writes + 600 reads/min | per application |
| 300 writes/min | per merchant, across applications |
| 10 test events/hour | per application |

Bulk work (backfills, exports of your own) should page with `limit: 100` and `for await`, run in a
background job, and honour `RateLimitError.retryAfterSeconds`. For CSV exports of payments, orders,
subscriptions or customers, the dashboard's Export button on each of those lists already does it.

## Caching the catalogue

Prices are **never edited** (only added or archived), so `fianto.prices.retrieve(id)` for a given id
can be cached for a long time; re-check its `status` (`ACTIVE` / `ARCHIVED`) when it matters, since an
archived price or product makes session creation fail (`price_archived`, `product_archived`). Product names and descriptions can be edited, so cache products
briefly. Most integrations keep price ids in config (`FIANTO_PRICE_PRO`) and never need to list them
at request time.

## Frontend

- Open checkout synchronously in the click handler (popup blockers); the SDK button does.
- `Cross-Origin-Opener-Policy: same-origin-allow-popups` on the page with the button, and serve it
  from the `success_url` origin.
- The browser packages need no provider or key: import `FiantoButton` only in the client component
  that renders it.

## Observability

- Log `err.requestId` (`X-Request-Id`, `req_…`) for every `APIError` and quote it to
  support@fianto.xyz. With raw HTTP (not the SDK) you can send your own `X-Request-Id` (8–64 of
  `[A-Za-z0-9_-]`) and fianto keeps it, joining your logs with fianto's.
- Wire `onError` (checkout route) and `onVerificationError` / `onError` (webhook route) to your error
  tracker: the browser sees most server-side failures (401, 403, 5xx, `merchant_token_account_missing`)
  only as 500 `internal_error`, and a wrong webhook secret only ever answers 400.
- Alert on: webhook 4xx/5xx rates, time since the last `order.paid`, reconcile finding events the
  webhook missed, the dashboard's **Needs attention** counts (past-due subscriptions, unmatched
  transfers, failed payments whose funds landed).
- Keep server clocks in sync (NTP): >300 s skew rejects every delivery.

## Go-live checklist

- [ ] Merchant approved; receiving wallet has a USDC token account
- [ ] Production env has `FIANTO_APP_ID`, `FIANTO_APP_SECRET`, `FIANTO_WEBHOOK_SECRET`; none in client bundles
- [ ] Webhook URL (exact final `https` URL, no redirect) shows **Verified**; `test.event` delivered
- [ ] Webhook handler dedupes on `event.id` in the fulfilment transaction and answers in < 10 s
- [ ] Fulfilment only on `order.paid` / `subscription.created`; `success_url` page says "confirming"
- [ ] Prices decided server-side; cancel routes check ownership
- [ ] `onError` / `onVerificationError` go to monitoring; reconcile job scheduled
- [ ] Button page: same origin as `success_url`, COOP `same-origin-allow-popups` if COOP is set
- [ ] Payer copy never says "not charged"; support path for refunds (from the merchant's wallet)
