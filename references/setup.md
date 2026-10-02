# Setup: from zero to working credentials

Live pages: https://docs.fianto.xyz/merchants/create-account.md,
`/merchants/{set-up-your-business,review-and-approval,developers-settings,products-and-prices}.md`,
`/get-started/do-i-need-a-developer.md`

The merchant does steps 1–4 (in the dashboard at merchant.fianto.xyz, plus one USDC transfer from a
wallet); an agent cannot do them. If a
developer's credentials are missing, walk them through the step they are on.

## Check what is already in place

```bash
# All three present in the server's environment?
node -e "for (const k of ['FIANTO_APP_ID','FIANTO_APP_SECRET','FIANTO_WEBHOOK_SECRET']) console.log(k, process.env[k] ? 'set' : 'MISSING')"

# Keys valid, and is the webhook URL verified? (the CLI reads the shell, so export the variables first)
npx @fianto/cli whoami
```

`whoami` prints the application and app id, the merchant's name, and the webhook status and URL (or
`not configured`). A 401 `invalid_api_credentials` means wrong, revoked or expired keys, keys from another deployment
(`FIANTO_BASE_URL`), a disabled application, or an account that is not approved and active.

## 1. Create the account

Email + password (8–30 chars, at least two of lowercase, uppercase, digits, symbols) → a 6-digit emailed code (valid 10 minutes, 5 tries). The account
exists only once the code is entered. Code emails are rate-limited and not guaranteed.

## 2. Set up the business (four steps on the setup page)

1. **Who gets paid**: a registered company, a business that is not registered, or "Just me", plus
   the profile. The display name becomes the name payers see on checkout once approved.
2. **Two-factor sign-in** (required, cannot be turned off): authenticator app + 10 recovery codes shown once.
3. **Receiving wallet**: sign a message (free, moves nothing) with a Solana wallet that can sign
   messages. One wallet cannot serve two merchants. **Choose a long-term wallet**: every subscription
   started while this is the receiving wallet pays it on chain until that subscription ends, even
   after a later wallet change.
4. **Submit for review**: details freeze; a submission cannot be withdrawn.

## 3. Review

| Status | Means |
|---|---|
| `SETUP` | Filling in the steps |
| `IN_REVIEW` | Submitted; details frozen. No review time is promised |
| `CHANGES_REQUESTED` | Read the note, update, submit again |
| `APPROVED` | Can create applications, products and take payments |
| `REJECTED` | Final; write to support@fianto.xyz if it is a mistake |

Until approval the dashboard refuses data, applications and products (`merchant_not_approved`), and
API keys answer 401. Decision emails are best effort; the setup page is the truth.

## 4. Before the first sale

- The receiving wallet needs a **USDC token account**: send it any amount of USDC once, or creating a
  checkout fails with 422 `merchant_token_account_missing`.
- **Create an application** (Developers → New application; password + 2FA code). Copy the app id
  (`fian_app_…`) and secret (`fian_sk_live_…`) straight away — the secret is shown once. Limits: 10
  active, 100 total.
- **Products and prices** (only for `price_id` sessions and subscriptions): Products → New product.
  Billing "One-time", "Monthly (every 30 days)" or "Yearly (every 365 days)". Prices are never
  edited: add a new one and archive the old. Copy the `fian_price_…` Public ID. Limits: 20 active
  prices per product, 200 prices per product, 1,000 products (nothing is ever deleted).
- **Webhook URL**: after the webhook route is deployed, Developers → application → Manage webhook →
  enter the `https` URL, password and code → Save and verify. Then "Reveal signing secret" →
  `FIANTO_WEBHOOK_SECRET` → restart → "Verify again" if the first check failed. A **Test** panel with
  "Send test event" appears once the URL is verified.

## Environment variables

| Variable | Where it comes from | Used by |
|---|---|---|
| `FIANTO_APP_ID` | Application page | `new Fianto()`, route helpers, CLI |
| `FIANTO_APP_SECRET` | Shown once at create / roll | same — server only |
| `FIANTO_WEBHOOK_SECRET` | "Reveal signing secret" (after a URL is set) | webhook handler, CLI `trigger`/`events tail`/`sign` |
| `FIANTO_BASE_URL` | Only for a self-hosted or local fianto backend | SDK, CLI |

Next.js reads `.env.local` itself. Node (Express, Hono) needs `node --env-file=.env` (Node ≥20.6) or
`dotenv`. Cloudflare Workers: Worker secrets, read from `c.env`. The CLI reads only exported shell
variables or flags.

## What needs code and what does not

The dashboard alone cannot sell: every checkout starts as a session the merchant's server creates.
Dashboard-only (no API): account and security, products and prices (the API only reads them),
customers, the Overview and Needs attention pages, CSV exports, webhook URL and redelivery, wallet
changes.
