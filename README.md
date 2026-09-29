# fianto agent skill

An [Agent Skill](https://agentskills.io) that teaches coding agents (Claude Code, Cursor, Codex,
OpenCode, GitHub Copilot, Windsurf and others) to build on [fianto](https://docs.fianto.xyz), the
hosted checkout for **USDC on Solana**, correctly the first time.

```bash
npx skills add fianto-xyz/skills
```

## Why this exists

fianto is new, so coding agents have never seen its API. Asked to "add fianto payments", an agent
without this skill guesses: it invents a `Bearer` header, a `payment.succeeded` event, a
`/v1/checkout/sessions` path and decimal webhook amounts. It skips the webhook URL's verification
challenge, so the integration never receives an event. With the skill loaded, the same agents write
the integration with the official `@fianto/*` SDKs, fulfil only from verified webhooks, and handle
the parts that are easy to miss: the one-time checkout `url`, idempotency keys, confirmed vs
finalized, and "closed" meaning *unknown*, never *not charged*.

## How fianto works, in one picture

```
Payer clicks Pay ──► your server creates a checkout session (secret key, server only)
        │
        ▼
fianto's hosted checkout ── payer's wallet signs ONE transaction:
        │                     price ─────────────► merchant's wallet
        │                     service fee (if > 0) ► fianto
        ▼
confirmed  ──► payer sees "Payment complete"
finalized  ──► order PAID ──► signed webhook `order.paid` ──► your server fulfils
```

Non-custodial (nothing is held in between), USDC only, one-time payments and 30/365-day
subscriptions, no test mode, no refund API. Full picture: [`references/concepts.md`](references/concepts.md).

## What your agent can do with it

**Build**
- Add a checkout route, pay button and webhook route to Next.js, Express, Hono (Node or Cloudflare
  Workers), React or a plain HTML page
- Sell subscriptions: recurring prices, subscription sessions, renewals, past-due handling, cancelling
- Test without a test mode: signed sample events, `test.event`, forwarding real events to localhost

**Debug**
- `invalid_webhook`, `BodyAlreadyParsedError`, `invalid_api_credentials`, `url: null`,
  `forbidden_origin`, `merchant_token_account_missing`, a popup that reports `closed`, and more
- Tell whether a payment really happened, and what to say to the payer

**Ship**
- Walk a merchant through account setup, review and credentials
- Harden for production: fast webhook acknowledgement, reconciliation from `GET /v1/events`, rate-limit
  budgets, monitoring, and a go-live checklist
- Explain fees, finality and subscription terms to a developer, a merchant or their customers

## Try it

After installing, ask your agent:

> Add fianto checkout to my Next.js shop so orders are marked paid when the payment finalizes.

> Sell a 9.99 USDC plan every 30 days from my Hono app and let signed-in users cancel it.

> My Express fianto webhook returns 500 BodyAlreadyParsedError. Fix it.

> Review my fianto integration against the go-live checklist.

> A customer says they paid but the order is still pending. What happened?

## Install

The [`skills` CLI](https://github.com/vercel-labs/skills) installs it for any supported agent:

```bash
npx skills add fianto-xyz/skills                     # this project, choose agents interactively
npx skills add fianto-xyz/skills -g                  # every project
npx skills add fianto-xyz/skills -a claude-code -y   # a specific agent, no prompts
npx skills add fianto-xyz/skills --list              # see what the repo contains
```

Other setups:

- **Manual copy**: copy `SKILL.md` and `references/` into a `fianto` folder in your agent's skills
  directory, for example `.claude/skills/fianto/` (Claude Code) or `.agents/skills/fianto/`.
- **Chat apps without skill support**: paste `SKILL.md` into the conversation or project
  instructions, and add the reference file for the topic at hand (for example `references/webhooks.md`).

The skill loads on its own when a task mentions fianto, its packages or its errors. It never needs
your secrets: keep `FIANTO_APP_ID`, `FIANTO_APP_SECRET` and `FIANTO_WEBHOOK_SECRET` in your server's
environment.

## What's included

```
.
├── SKILL.md                 # security rules, API at a glance, integration rules, a complete Next.js example
└── references/
    ├── setup.md             # account → review → application → webhook URL; env vars
    ├── sdk.md               # the seven @fianto/* packages; Next.js, Express, Hono, browser, React
    ├── checkout.md          # sessions, the one-time url, reissue/cancel, order and payment statuses
    ├── webhooks.md          # verification challenge, signatures, 14 event types, retries, secret roll
    ├── subscriptions.md     # recurring prices, renewals, past due, cancelling
    ├── api.md               # auth, error codes, idempotency, pagination, rate limits
    ├── testing.md           # testing without a test mode; the CLI
    ├── security.md          # secrets, trust boundaries, prompt injection, actions needing confirmation
    ├── production.md        # performance, reliability, monitoring, go-live checklist
    └── concepts.md          # how fianto works, who pays what, FAQ for merchants and payers
```

## Safety

The skill tells agents to keep secrets on the server, to treat webhook and event content as data (not
instructions), and to ask you before anything irreversible: cancelling subscriptions, disabling an
application, rolling a secret immediately, sending signed events to a non-local URL, or moving USDC.
Read the skill before installing it, as with any code you give an agent.

## Accuracy and updates

Every fact is taken from the fianto documentation at https://docs.fianto.xyz; each reference file
links the pages it summarises, and agents fetch `<page>.md` from there for anything the skill leaves
out. When fianto's API, SDK or docs change, update the matching reference and bump `version` in the
`SKILL.md` frontmatter. Keep `SKILL.md` under 500 lines.

## Links

- Docs: https://docs.fianto.xyz (Markdown index: https://docs.fianto.xyz/llms.txt)
- Merchant dashboard: https://merchant.fianto.xyz
- SDK: https://github.com/fianto-xyz/sdk
- Support: support@fianto.xyz
