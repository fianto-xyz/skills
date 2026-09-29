# Testing and the CLI

Live pages: https://docs.fianto.xyz/developers/testing.md, https://docs.fianto.xyz/developers/cli.md

**There is no test mode, no test keys and no sandbox.** Keys start `fian_sk_live_` on every
deployment. Each fianto deployment runs on one Solana cluster (mainnet-beta, devnet or localnet),
set by that backend itself: the deployment `FIANTO_BASE_URL` points at decides the cluster, not the
key prefix. There is no separate devnet API host to switch to.

## The four ways to test

1. **Signed local samples** — no API call, only the webhook secret:

   ```bash
   npx @fianto/cli trigger order.paid \
     --forward-to http://localhost:3000/api/webhooks/fianto \
     --secret "$FIANTO_WEBHOOK_SECRET"
   ```

   Prints `→ 200 order.paid (local sample)`. Works for any of the 14 event types
   (`subscription.created`, `order.expired`, …). Sample ids are fake (`sample_order_1001`).
   `--forward-to` must be `localhost` / `127.0.0.0/8` / `[::1]` unless `--allow-remote`. Non-2xx → exit 1.

2. **A real `test.event`** to the verified webhook URL (needs app id + secret; 10/hour per application):

   ```bash
   npx @fianto/cli trigger test.event
   # or: curl -X POST https://api.fianto.xyz/v1/webhook/test-event \
   #   --user "$FIANTO_APP_ID:$FIANTO_APP_SECRET" -H "Idempotency-Key: test-event-1"   → 202 { "event_id": "evt_…" }
   ```

   `test.event` is the only event fianto sends on request; business events (`order.paid`, …) are never
   sent on demand. No verified URL → 409 `webhook_endpoint_not_active`.

3. **Forward real events to localhost** (development aid, not delivery; once per event per run, no retry):

   ```bash
   npx @fianto/cli events tail --forward-to http://localhost:3000/api/webhooks/fianto --since 5m
   ```

4. **Real payments without mainnet funds**: point the SDK/CLI at a self-hosted or local fianto backend
   that runs on devnet or localnet, with
   an application created on that deployment: `export FIANTO_BASE_URL=http://localhost:3000` (https
   except `localhost`, `127.0.0.1`, `[::1]`; no `/v1` suffix). Keys from another deployment → 401 `invalid_api_credentials`.

In unit tests, sign fixtures yourself:

```ts
import { sampleEvent, signWebhook } from '@fianto/sdk/webhooks';

const { body, headers } = await signWebhook({ event: sampleEvent('order.paid'), secret: 'whsec_...' });
await fetch('http://localhost:3000/api/webhooks/fianto', { method: 'POST', headers, body });
```

`sampleVerificationEvent()` builds the URL-check request.

## CLI reference

The CLI reads the **shell environment, not `.env`** — export the variables or pass flags (flags win).

| Flag | Env | Needed by |
|---|---|---|
| `--app-id` | `FIANTO_APP_ID` | `whoami`, `events …`, `trigger test.event` |
| `--app-secret` | `FIANTO_APP_SECRET` | same |
| `--base-url` | `FIANTO_BASE_URL` | optional |
| `--secret` / `--secret-file` | `FIANTO_WEBHOOK_SECRET` | `events tail`, `trigger --forward-to`, `sign` |

| Command | Does |
|---|---|
| `fianto whoami` | Which application/merchant the keys belong to, webhook URL + status |
| `fianto events list [--type T] [--limit N]` | One page of events, newest first |
| `fianto events get <evt_id>` | One event as JSON |
| `fianto events tail --forward-to URL [--since 5m] [--type T] [--interval 2000]` | Poll + re-sign + forward |
| `fianto trigger <type> --forward-to URL` | Signed local sample |
| `fianto trigger test.event` | Real delivery to the verified URL |
| `fianto sign --payload file.json [--id] [--timestamp]` | Sign a body and print a ready curl command |

Exit codes: 0 ok, 1 handled error (API errors print `code: message (request req_…)`), 2 usage error,
130 Ctrl-C on `events tail`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Route answers 400 `invalid_webhook` to samples | CLI and route use different `whsec_` secrets |
| `trigger order.paid` without `--forward-to` exits 2 | Only `test.event` goes through fianto |
| 409 `webhook_endpoint_not_active` | Set and verify the webhook URL first |
| 401 against a self-hosted/local backend | Keys belong to another deployment, or `FIANTO_BASE_URL` unset |
| CLI says a variable is missing though it's in `.env` | Export it in the shell |
