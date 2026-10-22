# How fianto works (for explaining to people)

Live pages: https://docs.fianto.xyz/get-started/what-is-fianto.md,
`/get-started/how-money-moves.md`, `/concepts/{finality,fees,payment-lifecycle,subscription-lifecycle,subscriptions-program}.md`,
`/payers/{paying-with-fianto,what-it-costs,your-subscriptions,get-help}.md`,
`/merchants/{shop,payment-links}.md`, `/developers/{shop-sales,payment-link-sales}.md`

Use this to answer "how does it work?", "who pays what?" and "why hasn't my order updated?" for
developers, merchants and their customers (payers).

## The people involved

| Who | Does |
|---|---|
| **Merchant** | Owns the fianto account, the receiving wallet, products and prices (dashboard at merchant.fianto.xyz) |
| **Developer** | Writes the three pieces: checkout route, pay button, webhook route |
| **Payer** | Pays on fianto's hosted checkout page from their own Solana wallet; manages subscriptions at fianto.xyz/my |
| **fianto** | Hosts checkout, broadcasts the payer's signed transaction, follows it to finality, sends webhooks, charges subscription renewals |

## A one-time payment, end to end

```
Payer clicks Pay ──► your server: POST /v1/checkout-sessions ──► { id, url }
        │
        ▼
fianto checkout page ── payer's wallet signs ONE transaction:
        │                  price ──► merchant's wallet
        │                  service fee (if > 0) ──► fianto treasury
        ▼
fianto broadcasts it ──► CONFIRMED  (payer sees "Payment complete")
                     ──► FINALIZED  (order PAID, webhook order.paid ──► your server fulfils)
```

- **Non-custodial.** The money goes straight from payer to merchant in one transaction; nothing is
  held in between. fianto only broadcasts what the payer signed.
- **USDC on Solana only.** No other currencies or chains.
- **Confirmed vs finalized.** The payer is shown success at *confirmed*; the merchant is told only at
  *finalized*, because a confirmed block can still be rolled back and a finalized one cannot. This
  gap is why a success page must say "confirming" and why fulfilment waits for `order.paid`.

## Who pays what

| When | Price (USDC) | Service fee (USDC) | Solana network fee (SOL) | Account deposit (SOL) |
|---|---|---|---|---|
| One-time payment | Payer → merchant | Payer, when > 0 | fianto | None |
| Starting a subscription | Payer, first period | Payer, when > 0 | fianto | fianto |
| Each renewal | Pulled from payer's wallet | Pulled with it, when > 0 | fianto | None |

Payers need only USDC, no SOL. fianto closes ended subscriptions itself; there is no reclaim step.

- The merchant receives the **full price**; fianto's service fee (capped at 10 %, rounded down in
  the payer's favour) is **added on top** and paid by the payer. The checkout page always shows a
  "Service fee" line ("None" when zero). The fee is fixed when the session is created.
- In the API: `amount` = price, `fee_amount` = fee, `total_amount` = what the payer paid (base units).
- The payer needs an existing, unfrozen USDC token account, and enough USDC (no SOL); the
  wallet must be able to sign a transaction without sending it.

## Subscriptions

- Every 30 days (`MONTH`) or 365 days (`YEAR`) — days, not calendar months.
- The payer signs once to subscribe (first period charged in that transaction). After that, fianto's
  charging key takes each renewal within limits the Solana Subscriptions program enforces: at most
  the plan amount per period, only to the plan's destinations, nothing after the subscription is
  cancelled on Solana or expired.
- Terms are fixed on chain: amount and interval never change for an existing subscriber, and they
  keep paying the wallet and fee they subscribed with.
- Each renewal is tried 2 min after the period ends, then 1 h, 6 h and 24 h after it; the 4th
  counted failure (insufficient funds, revoked approval, frozen or closed account) ends it. Missed periods are never billed later.
- Payers see and cancel their subscriptions at **fianto.xyz/my** (sign in with the wallet; free).
  A payer's cancel takes effect at period end. A merchant's cancel is final and does not end the
  on-chain subscription.

## Selling without code: shops and payment links

A merchant can also sell from the dashboard, with no integration:

- a **shop**, a page of their own at `https://<name>.fianto.xyz`;
- **payment links**, made under **Payments → Payment links**: a reusable link
  `https://checkout.fianto.xyz/l/<16 characters>` that sells one catalog price (one-time or recurring).

Both are dashboard-only: no v1 endpoint, SDK call or API key creates, lists or reads them. Their checkout
sessions belong to the merchant's **"Shop" application**, made by the first shop or the first Pay on a payment
link. It has no secret, so v1 never returns these sales; they reach code only through that application's
webhook URL (settable in Developers), with the usual event types. Tell them apart:

| Sale | `order_id` | `metadata` |
|---|---|---|
| Shop | `shop_<uuid>` | `{ "source": "shop", "shop_slug": "<name>" }` |
| Payment link | `plink_<uuid>` | `{ "source": "payment_link", "payment_link": "<the link's 16-character code>" }` |

A handler on the merchant's own application never receives them. Payment links send the payer no email.

## What fianto is not

- Not a custodian, and there is **no refund API**: refunds are USDC the merchant sends from their own
  wallet.
- **No test keys.** Devnet is a sandbox with test USDC and the same keys: `network: 'devnet'` / `FIANTO_NETWORK=devnet`. Other testing: signed samples, `test.event`, self-hosted/local backends.
- Not usable before review: every merchant account is approved by an admin first.
- fianto sends payers **no emails or receipts**; the email a payer enters goes to the shop.

## Common questions, short answers

| Question | Answer |
|---|---|
| "The payer saw success but my order isn't paid" | Normal for a few moments: success shows at confirmed, `order.paid` comes at finalized. If it stays pending, check the webhook URL is verified and read the order with `retrieveByOrderId` |
| "The popup closed — did they pay?" | Unknown. Check the order; never tell them they weren't charged |
| "Can I refund through fianto?" | No. Send USDC from the merchant wallet to the payer's wallet |
| "Why is the payer charged more than my price?" | The service fee is added on top (when above zero), and every subscription charge, the first included, adds the plan's flat network fee in USDC (currently 0.01 USDC); fianto pays the Solana network fee and account deposits |
| "Can I change a subscriber's price?" | No; add a new price for new subscribers. Existing ones keep their terms |
| "The payer paid twice" | `order.duplicate_payment`; the USDC is in the merchant's wallet — refund by hand |
| "Can I test without real money?" | Yes, on devnet: `new Fianto({ network: 'devnet' })` or `FIANTO_NETWORK=devnet`, same keys, test USDC (faucet link on the checkout page). Approved merchants only. Also `fianto trigger` samples or a self-hosted/local backend |
| "Can I create a payment link through the API?" | No. Payment links are made in the dashboard only; from code, create a checkout session |
| "Webhooks arrive with `order_id` `plink_…` or `shop_…`" | Payment-link or shop sales, sent to the merchant's Shop application endpoint; read `metadata.source` |
| "Who does the payer contact?" | The shop, for payments and refunds; support@fianto.xyz for the checkout page or portal |

## Ids at a glance

`fian_app_` application · `fian_sk_live_` app secret · `whsec_` webhook secret · `fian_cs_` checkout
session · `fian_ord_` order · `fian_pay_` payment · `fian_cus_` customer · `fian_sub_` subscription ·
`fian_plan_` on-chain plan · `fian_price_` price · `evt_` event · `req_` request id. `order_id` is
always the merchant's own id.
