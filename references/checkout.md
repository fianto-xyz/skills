# Checkout sessions (one-time payments)

Live pages: https://docs.fianto.xyz/developers/checkout-sessions.md,
https://docs.fianto.xyz/concepts/payment-lifecycle.md, https://docs.fianto.xyz/concepts/finality.md

## Create: `POST /v1/checkout-sessions`

SDK: `fianto.checkoutSessions.create(params, { idempotencyKey? })`.

| Field | Rules |
|---|---|
| `mode` | `payment` (here) or `subscription` (see subscriptions.md). Required |
| `order_id` | Your own id, up to 128 chars. Required |
| `price_id` | An active ONE-TIME price on an active product (`fian_price_…`). Send this **or** `amount` |
| `amount` + `description` | Decimal USDC **string**, `"0.01"` to `"1000000"`, max 6 decimals (`"12.50"`). `description` (≤1,000 chars) is required with `amount` |
| `success_url`, `cancel_url` | `https` only, ≤2,048 chars. Required |
| `ui_mode` | `redirect` (default) or `popup` |
| `expires_at` | 5 minutes to 24 hours from now. Default 30 minutes |
| `customer_email`, `customer_reference` | Your own details about the payer |
| `metadata` | ≤20 keys; key ≤40 chars, value ≤500 |
| `line_items` | ≤50, display only: fianto never sums or checks them against the amount |

Unknown fields → 400 `validation_failed` (`field` names the first one). A JSON number for `amount`
also fails `validation_failed`: send a string.

Response: the session object with `id` (`fian_cs_…`), `url`, `status`, `expires_at`, …

### The `url` comes back only once

`url` is returned only by **create** and by **reissue**. Every read returns `url: null`. Creating again
with the same `order_id` while its session is still `OPEN` returns that same session with
`url: null`. So:

- store `session.id` (and send the payer to `url`) straight after create;
- retry a failed create with the **same** `Idempotency-Key` — the replay returns the stored response,
  `url` included (see api.md);
- to get a fresh link for an open session: `POST /v1/checkout-sessions/{id}/link`
  (`fianto.checkoutSessions.reissueLink(id)`). The old link stops working.

The SDK route helpers (`Checkout()` / `checkout()` / `createCheckoutHandler`) handle this for you:
same `order_id` + same terms → they reissue; different terms → 409 `order_session_mismatch` (give
changed terms a new `order_id`, e.g. include a cart version).

## Send the payer to checkout

- **Redirect** (`ui_mode: redirect`): send the browser to `url`. After success, checkout moves the
  payer to `success_url` after ~3 s; cancel → `cancel_url`. fianto adds **no query parameters** to
  either URL.
- **Popup** (`ui_mode: popup`): open with `@fianto/js` / `@fianto/react`. The popup posts
  `{ type: 'fianto.checkout', session_id, status }` only to the **origin of `success_url`**, then
  closes — serve the page that opens checkout from that origin. SDK route helpers always create popup
  sessions.

## Fulfil from fianto, never from the browser

The checkout page shows the payer success at **confirmed**. fianto marks the order `PAID`, the
payment `SUCCEEDED`, and sends `order.paid` (+ `checkout.session.completed` if the session was still
`OPEN`) only at **finalized**. Reaching `success_url` or a popup `succeeded` proves nothing. Fulfil
from the verified `order.paid` webhook, or from a server-side read showing the order `PAID`
(`fianto.orders.retrieveByOrderId(orderId)` → `GET /v1/orders/lookup?order_id=`). Make the
`success_url` page say "we're confirming your payment".

## Reissue and cancel

- **Reissue** `POST /v1/checkout-sessions/{id}/link`. Refused with 409 `session_not_reissuable` (not
  `OPEN`, or <120 s left), 409 `payment_in_progress` (a payment is in progress or the last prepared
  transaction could still land), 409 `checkout_unavailable` (+ `Retry-After`; SDK retries it).
- **Cancel** `POST /v1/checkout-sessions/{id}/cancel` (`fianto.checkoutSessions.cancel(id)`). Only an
  `OPEN` session with no payment in progress (else 409 `session_not_cancelable` /
  `payment_in_progress`). Sends `checkout.session.canceled`; **no order event** — the order stays
  `PENDING`. The payer can cancel on the checkout page with the same result.

Both are POSTs → need an `Idempotency-Key` (the SDK adds one).

## Statuses

**Session** (both modes): `OPEN` → `COMPLETED` | `EXPIRED` | `CANCELED`.

**Order** (payment mode only; subscription sessions create no order): `PENDING`, `PAID`, `EXPIRED`.

- `PENDING → PAID` when a payment finalizes; `PENDING → EXPIRED` when its session expires.
- `EXPIRED → PAID` when a late payment still settles. **`EXPIRED` is not final.**
- `EXPIRED → PENDING` when you create a new session with the same `order_id` (the order takes the new
  terms). A cancelled session's order is reused the same way. There is no `CANCELED` order status.
- Only a `PAID` order refuses a new session: 409 `order_already_paid`.

**Payment**: `PROCESSING → CONFIRMED → SUCCEEDED`; `PROCESSING`/`CONFIRMED → FAILED`;
`FAILED → PROCESSING` if the transfer lands after all. **`SUCCEEDED` is final; `FAILED` is not.**
A payment whose session was superseded (a newer session for the same order, or changed order terms)
is recorded `FAILED` at finalized even though its funds landed — it never shows `SUCCEEDED` first.

A `FAILED` payment can have its USDC in the merchant's wallet. Never tell a payer "you were not
charged" from a status alone; check the payment first. There is no refund API: refunds are sent by the
merchant from their own wallet.

Sweeps: every 30 s fianto expires lapsed `OPEN` sessions (a payment in flight blocks expiry) → sends
`checkout.session.expired` + `order.expired`. If instead a new create replaces a lapsed session, it
sends `checkout.session.expired` but **no** `order.expired` (the order is reopened). Every 60 s fianto
also scans for transfers carrying a session's reference (for 24 h after it closes); such payments
start at `CONFIRMED`. A second transfer for an already `PAID` order → `order.duplicate_payment`
(fianto records up to 20 unmatched transfers per session).

A payment that finalizes after its session `EXPIRED`/`CANCELED` still pays the order (`order.paid`,
no `checkout.session.completed`) — unless a newer session was opened for that order, in which case
the payment is `FAILED` with funds landed. Keep the webhook route listening.

## Payer-side limits you may see reported

- The checkout page refuses `build_limit_reached` (10 successful builds / 30 attempts per session) or
  `too_close_to_expiry` (no build in the last 120 s): cancel and create a new session with the same
  `order_id`.
- The payer needs an existing, unfrozen USDC token account, the price + fee in USDC, and ≥0.00001 SOL.
  A one-time payment costs the payer no rent.
- The merchant's receiving wallet must already have a USDC token account, or create fails with 422
  `merchant_token_account_missing` (fix: send that wallet any amount of USDC once). fianto never creates
  token accounts.

## Checkout errors

| Error | Cause → fix |
|---|---|
| 400 `amount_or_price_required` | Neither/both of `price_id`/`amount`, or `amount` without `description` |
| 400 `amount_out_of_range` | Outside 0.01–1,000,000 |
| 400 `validation_failed` `field: "amount"` | Not a decimal string with ≤6 decimals |
| 400 `url_insecure` | An `http` success/cancel URL → use `https` |
| 400 `expires_at_out_of_range` | Not 5 min–24 h away |
| `price_not_found` / `price_archived` / `product_archived` / `price_type_mismatch` | `price_id` is not an active one-time price on an active product |
| 409 `order_already_paid` | This `order_id` is paid → new `order_id` to charge again |
| 422 `merchant_token_account_missing` | Receiving wallet has no USDC token account |
| `url` is `null` | Order already has an open session, or you read instead of created → reissue |
| 409 `session_not_reissuable` | Not open / <120 s left → cancel if open, then create again with same `order_id` |
| 409 `payment_in_progress` | Wait; do not start a second payment for the order |
| 409 `checkout_unavailable` | fianto couldn't check Solana in time → retry after `Retry-After` (5 s) |
