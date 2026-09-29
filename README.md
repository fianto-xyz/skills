# fianto agent skill

An [Agent Skill](https://agentskills.io) that teaches coding agents (Claude Code, Cursor, Codex,
OpenCode, GitHub Copilot and others) to integrate [fianto](https://docs.fianto.xyz), the hosted USDC-on-Solana
checkout, correctly: checkout sessions, subscriptions, webhooks and the `@fianto/*` SDKs.

## Install

```bash
npx skills add fianto-xyz/skills
```

Options: `--skill fianto` installs only this skill, `-g` installs it for every project,
`-a claude-code` (or another agent) picks the agent, `--list` shows what the repo contains.
Source: https://github.com/fianto-xyz/skills

Then ask your agent something like "add fianto checkout to my Next.js app" or "why does my fianto
webhook answer invalid_webhook?". The skill loads when a task mentions fianto or its packages.

## What it covers

| File | Covers |
|---|---|
| `skills/fianto/SKILL.md` | The integration rules, a complete Next.js example, and corrections for common wrong assumptions |
| `references/sdk.md` | All seven packages; Next.js, Express, Hono (Node and Workers), browser and React wiring |
| `references/checkout.md` | Session fields, the one-time `url`, reissue/cancel, order and payment statuses |
| `references/webhooks.md` | Verification challenge, signatures, the 14 event types, delivery and retries |
| `references/subscriptions.md` | Recurring prices, subscription sessions, renewals, cancelling |
| `references/api.md` | Auth, error codes, idempotency, pagination, rate limits |
| `references/testing.md` | Testing without a test mode, and the CLI |

Everything here is taken from the fianto documentation at https://docs.fianto.xyz. Each reference
links the pages it summarises; agents fetch `<page>.md` from there for anything the skill leaves out.

## Updating

When fianto's API, SDK or docs change, update the matching reference file and bump `version` in the
`SKILL.md` frontmatter. Keep `SKILL.md` under 500 lines; put detail in `references/`.
