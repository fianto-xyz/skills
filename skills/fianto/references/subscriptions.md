# Subscriptions

Live pages: https://docs.fianto.xyz/developers/subscriptions.md,
https://docs.fianto.xyz/get-started/subscriptions-quickstart.md,
https://docs.fianto.xyz/concepts/subscription-lifecycle.md,
https://docs.fianto.xyz/concepts/subscriptions-program.md

A subscription charges one payer wallet the same USDC price every **30 days** (`MONTH`) or **365
days** (`YEAR`) — counted in days, not calendar months, and there are no other intervals. It runs on the Solana
Subscriptions program (`De1egAFMkMWZSN5rYXRj9CAdheBamobVNubTsi9avR44`). Subscriptions need the API:
the dashboard alone cannot sell them.

## 1. Recurring price (dashboard only)

Create a product and a recurring price in the dashboard; copy its `fian_price_…` id (e.g. into
`FIANTO_PRICE_PRO`). The v1 API can **read** products/prices (`fianto.products.list()`,
`fianto.prices.list()`) but never create or edit them. Prices are never edited: add a new one and
archive the old.

## 2. Subscription session

```ts
import { Checkout } from '@fianto/nextjs';

export const POST = Checkout({
  createSession: async () => {
    const priceId = process.env.FIANTO_PRICE_PRO;
    if (!priceId) return new Response('Plan not configured', { status: 500 });
    return {
      mode: 'subscription',
      order_id: `sub_${crypto.randomUUID()}`, // your own id for this sign-up
      price_id: priceId, // required; NO amount, NO description
      success_url: 'https://shop.example/welcome',
      cancel_url: 'https://shop.example/pricing',
    };
  },
  onError: (error) => console.error(error),
});
```

Put the signed-in user's id somewhere you will get back: `customer_reference` (echoed as
`data.customer.reference`) and/or `metadata`, and keep your own `order_id` → user mapping.

Errors: 400 `price_id_required`; 400 `amount_not_allowed` (you sent `amount` or `description`);
422 `price_not_recurring`; 429 `plan_limit_reached` (50 new plans per merchant in any 24 hours).

The `url`/reissue/cancel/redirect/popup rules are the same as for one-time sessions (checkout.md).
Use `label="subscribe"` on the button.

**A subscription session creates no order** — no `order.*` events. Follow it only through
`subscription.*` events and reads.

## 3. Plans (on-chain, automatic)

The first subscription session for a price creates its plan on Solana; fianto signs and pays for
it. Until it is on chain, the hosted page shows the plan as preparing (no pay button). Failed plan
creation retries after 30 s, 2 min, 10 min, 1 h, then fails (`subscription_plan_failed` on the
checkout page → create a new subscription session for the price to start a fresh attempt).

A plan is tied to price + receiving wallet + fee + treasury. If the merchant's wallet or the fee
changes, **new** subscribers get a new plan; existing subscribers keep paying the old wallet and fee.
An open subscription session whose link went out of date is cancelled by fianto
(`checkout.session.canceled`, payer sees `checkout_link_outdated`).

## 4. Follow it through webhooks

| Event | Do |
|---|---|
| `subscription.created` | Grant access (subscribe tx finalized, first period already paid) |
| `subscription.renewed` | Extend access to `data.current_period_end` |
| `subscription.payment_failed` | A counted failure; `data.failure.{reason, attempt_no, next_attempt_at}` |
| `subscription.past_due` | First counted failure — warn the user; fianto keeps retrying |
| `subscription.cancel_scheduled` | Ends at period end; `cancel_reason` says who |
| `subscription.cancel_withdrawn` | Payer resumed; keep access |
| `subscription.ended` | Remove access; `end_reason` `MERCHANT_CANCELED` / `PAYER_CANCELED` / `PAYMENT_FAILED` (the type also lists `PLAN_ENDED`/`PLAN_SUNSET`, unused today — treat any unknown reason as ended) |

Statuses: `ACTIVE`, `PAST_DUE`, `ENDED` (final). The same dedupe/at-least-once/unordered rules as
every webhook apply (webhooks.md).

## Renewals

- fianto's charging key signs each renewal and pays its network fee; the payer pays price + service fee.
- Attempts: 2 min after the period ends, then 1 h, 6 h, 24 h after it.
- **Counted** failures: `INSUFFICIENT_FUNDS`, `DELEGATION_REVOKED`, `ACCOUNT_FROZEN`, `ACCOUNT_CLOSED`.
  Other failures retry every 5 min without using an attempt — except a renewal refused because the
  payer already cancelled on Solana, which is recorded as the payer's cancel and not retried.
- 1st counted failure → `PAST_DUE` (`payment_failed` + `past_due`); 2nd/3rd → `payment_failed`;
  success → `ACTIVE` (`renewed`); **4th** → `ENDED`, `end_reason: PAYMENT_FAILED` (`payment_failed` +
  `ended`).
- No back-billing: a missed period is never charged later.

## 5. Cancel from your server

```ts
import { Fianto } from '@fianto/sdk';
const fianto = new Fianto();

await fianto.subscriptions.cancel(subscriptionId, { at: 'period_end' }); // or { at: 'now' }
```

- `at: 'now'` → ends at once, `subscription.ended` (`MERCHANT_CANCELED`).
- `at: 'period_end'` → `cancel_at_period_end: true`, `subscription.cancel_scheduled`. Asking again
  while scheduled sends no second event; if the payer scheduled it, your request still sets
  `merchant_cancel_requested: true` (`cancel_reason` stays `PAYER_CANCELED`), so your cancel stands
  even if the payer resumes.
- `at` is required (`validation_failed` `field: "at"` otherwise). 409 `subscription_already_ended` if
  already ended.
- Authorise it yourself: only cancel a subscription id that belongs to the signed-in user (look it up
  from your own records, not from the request body).

**A merchant cancel is final and changes nothing on Solana.** It only stops fianto charging; it
cannot be undone; the payer's on-chain subscription stays live; with `now` a renewal already in
flight can still land. There is no refund API — refunds are sent manually from the merchant's
wallet. The same wallet cannot re-subscribe to the same plan until the payer uses "Cancel on Solana"
in the payer portal, waits for the on-chain end and reclaims the rent.

Payers can also cancel themselves (payer portal or another wallet app): ends at period end,
`subscription.cancel_scheduled` with `PAYER_CANCELED` (picked up within ~5 min if made outside the
portal).

## Money

When subscribing the payer pays the first period's price + service fee (when >0) + network fee +
rent for the accounts the subscribe transaction creates. Each renewal pulls price + fee. The shared
USDC approval is one per wallet and mint, across every Subscriptions-program subscription: never tell
a payer to revoke it to stop one subscription.
