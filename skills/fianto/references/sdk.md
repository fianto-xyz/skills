# SDK packages and framework wiring

Live pages: https://docs.fianto.xyz/developers/sdks/overview.md and the per-package pages
`/developers/sdks/{sdk,nextjs,express,hono,js,react}.md`

All packages are TypeScript, ESM + CommonJS (CLI: ESM only), Node.js ≥ 20.3.

| Package | For | Runs on | Peer |
|---|---|---|---|
| `@fianto/sdk` | API client; `/webhooks` (verify/sign); `/handlers` (framework-free route handlers) | server | — |
| `@fianto/nextjs` | `Checkout()`, `Webhooks()` App Router handlers | server | `next` ≥15 |
| `@fianto/express` | `checkout()`, `webhooks()` middleware | server (Node) | `express` ≥4 |
| `@fianto/hono` | `checkout()`, `webhooks()` handlers | server (Node, Workers, Bun, Deno*) | `hono` ≥4 |
| `@fianto/js` | `openCheckout`, `redirectToCheckout`, `fetchCheckoutSession`, `focusCheckout`, `<fianto-button>` | browser | — |
| `@fianto/react` | `FiantoButton`, `useCheckout` (re-exports `fetchCheckoutSession` + errors) | browser | `react` ≥19 |
| `@fianto/cli` | `fianto` command (see testing.md) | dev machine / CI | — |

\*CI tests on Node.js 22 and 24 and smoke-tests on 20.3.0; the other runtimes are designed for but untested.

Install per stack:

```bash
# Next.js
npm install @fianto/nextjs @fianto/react @fianto/sdk
# Express
npm install @fianto/express @fianto/js @fianto/sdk
# Hono
npm install @fianto/hono @fianto/js @fianto/sdk
# Any other server
npm install @fianto/sdk
```

## `Fianto` client

```ts
import { Fianto } from '@fianto/sdk';
const fianto = new Fianto(); // FIANTO_APP_ID, FIANTO_APP_SECRET, optional FIANTO_BASE_URL
// or: new Fianto({ appId, appSecret, baseUrl?, timeoutMs?, maxRetries?, fetch? })
```

| Option | Env | Default |
|---|---|---|
| `appId` | `FIANTO_APP_ID` | required (`fian_app_…`) |
| `appSecret` | `FIANTO_APP_SECRET` | required (`fian_sk_live_…`), server only |
| `baseUrl` | `FIANTO_BASE_URL` | `https://api.fianto.xyz` (https except `localhost`, `127.0.0.1`, `[::1]`; **without** `/v1`) |
| `timeoutMs` | — | 30000 per attempt |
| `maxRetries` | — | 2 (0–10) |
| `fetch` | — | global `fetch` |
| `dangerouslyAllowBrowser` | — | `false`; `new Fianto()` throws in a browser. Never set it: there is no publishable key |

Missing `appId`/`appSecret` throws at construction ("Missing appId: pass it to new Fianto({ appId })
or set FIANTO_APP_ID.").

Resources (params/fields are the API's snake_case):

```
fianto.application.retrieve()
fianto.checkoutSessions.create(params, opts?) / retrieve(id) / cancel(id) / reissueLink(id)
fianto.orders.retrieve(id) / list(params?) / retrieveByOrderId(orderId)
fianto.payments.retrieve(id) / list(params?)
fianto.subscriptions.retrieve(id) / list(params?) / cancel(id, { at: 'now' | 'period_end' })
fianto.products.retrieve(id) / list(params?)     // read-only
fianto.prices.retrieve(id) / list(params?)       // read-only
fianto.events.retrieve(id) / list(params?)
fianto.webhookEndpoint.sendTestEvent()
```

Per-call options (last argument): `idempotencyKey`, `timeoutMs`, `signal`, `maxRetries`. For
`retrieve`, `checkoutSessions.cancel`, `reissueLink`, `application.retrieve`, `sendTestEvent` the
options come **after** an empty params object: `fianto.checkoutSessions.cancel(id, {}, { idempotencyKey })`.

`list()` returns a `PagePromise`: `await` → `{ items, next_cursor }`; `for await (const x of
fianto.orders.list({ limit: 100 }))` walks every item.

Amounts helper — no floating point:

```ts
import { usdc } from '@fianto/sdk';
usdc.toBaseUnits('12.5');                    // '12500000'
usdc.fromBaseUnits('12500000');              // '12.5'
usdc.format('12500000');                     // '12.50 USDC'
usdc.format('12500000', { symbol: false });  // '12.50'
```

## Checkout route handler (`Checkout()` / `checkout()` / `createCheckoutHandler`)

| Option | Notes |
|---|---|
| `createSession(request, context)` | Required. Return session params decided **on the server**, or a `Response` to refuse. `ui_mode` is not accepted: always popup. `context` = Next route context / Express `{ req, res }` / Hono `c` |
| `fianto` | A client; default `new Fianto()` from env on first request (pass one on Workers) |
| `allowedOrigins` | Allowed `Origin`s when the browser doesn't send `Sec-Fetch-Site: same-origin` (needed behind proxies) |
| `onError(error)` | Receives the original error for every failure. **Always wire it** |

Only `POST` (else 405 `method_not_allowed`), and only same-origin: `Sec-Fetch-Site: same-origin` or an
`Origin` in `allowedOrigins` (else 403 `forbidden_origin`; a sibling subdomain is refused too). Answers 200 `{ id, url }`.
Same `order_id` with an open session and the same terms → reissues its link; different terms → 409
`order_session_mismatch`.

Errors reach the browser as `{ error: { code, message } }` only for an allowlist (`payment_in_progress`,
`order_already_paid`, `checkout_unavailable`, `rate_limited`, `session_not_reissuable`,
`subscription_preparing`, `plan_limit_reached`, `order_session_mismatch`, and param validation codes).
Everything else — 401 (wrong keys, or an account not approved and active), any 403, 5xx,
`merchant_token_account_missing` —
reaches the browser as 500 `internal_error`; only `onError` shows the real cause.

## Next.js (App Router)

```ts
// app/api/checkout/route.ts
import { Checkout } from '@fianto/nextjs';

const PRICES: Record<string, { amount: string; description: string }> = {
  beans: { amount: '10.00', description: 'Coffee beans, 1 kg' },
};

export const POST = Checkout({
  createSession: async (request) => {
    const body = (await request.json().catch(() => null)) as { item?: string } | null;
    const price = body?.item ? PRICES[body.item] : undefined;
    if (!price) return new Response('Unknown item', { status: 400 });
    return {
      mode: 'payment',
      order_id: `order_${crypto.randomUUID()}`, // use your own order's id
      amount: price.amount,
      description: price.description,
      success_url: 'https://shop.example/thank-you',
      cancel_url: 'https://shop.example/cart',
    };
  },
  onError: (error) => console.error('fianto checkout failed', error),
});
```

Webhook route: see SKILL.md / webhooks.md. `.env.local` is read by Next automatically. Components
using `FiantoButton` / `useCheckout` must be client components (`'use client'`).

## Express

```ts
import express from 'express';
import { checkout, webhooks } from '@fianto/express';

const app = express();
app.post('/api/checkout', checkout({ createSession: async (request, { req }) => ({ /* … */ }), onError: console.error }));
app.post('/webhooks/fianto', webhooks({ onOrderPaid: async (event) => { /* … */ } }));
app.use(express.json()); // AFTER the fianto routes, or scoped: app.use('/api/other', express.json())
app.listen(3000);
```

- Both fianto routes need the untouched body → mount **before** any body parser. `express.raw()` on
  the route does not rescue a route behind a global parser (`BodyAlreadyParsedError`, 500).
- `createSession` gets a fetch `Request` plus `{ req, res }` (so `req.user` is available). Never send
  through `res`.
- Body cap 1 MiB (413 `payload_too_large`).
- Behind a TLS-terminating proxy: `app.set('trust proxy', 1)` and/or `allowedOrigins:
  ['https://shop.example']`, else 403 `forbidden_origin`.
- Node doesn't read `.env`: `node --env-file=.env server.js` (Node ≥20.6) or `dotenv`.

## Hono

On Node: same as Express shape with `app.post('/api/checkout', checkout({...}))`, start with
`@hono/node-server`'s `serve({ fetch: app.fetch, port: 3000 })`; `export default app` alone starts
nothing on Node.

On Cloudflare Workers (no `process.env`) build handlers **inside** the route:

```ts
import { Hono } from 'hono';
import { Fianto } from '@fianto/sdk';
import { checkout, webhooks } from '@fianto/hono';

type Bindings = { FIANTO_APP_ID: string; FIANTO_APP_SECRET: string; FIANTO_WEBHOOK_SECRET: string };
const app = new Hono<{ Bindings: Bindings }>();

app.post('/api/checkout', (c) =>
  checkout({
    fianto: new Fianto({ appId: c.env.FIANTO_APP_ID, appSecret: c.env.FIANTO_APP_SECRET }),
    createSession: async () => ({ /* … */ }),
    onError: (error) => console.error(error),
  })(c),
);
app.post('/webhooks/fianto', (c) =>
  webhooks({ secret: c.env.FIANTO_WEBHOOK_SECRET, onOrderPaid: async (event) => { /* … */ } })(c),
);
export default app;
```

## Browser: `@fianto/js`

```ts
import { fetchCheckoutSession, openCheckout } from '@fianto/js';

const status = document.querySelector('#status')!; // e.g. <p id="status" role="status">

document.querySelector('#pay')?.addEventListener('click', () => {
  // Call synchronously in the click handler — no await/setTimeout before it (popup blockers).
  openCheckout({
    session: () => fetchCheckoutSession('/api/checkout', { body: { item: 'beans' } }),
    fallback: 'redirect', // default: popup blocked → full-page redirect
    }).then(
    (result) => { status.textContent = MESSAGES[result.status] ?? ''; }, // MESSAGES below
    (error) => console.error(error),
  );
});
```

- `fetchCheckoutSession(endpoint, { body })` POSTs JSON same-origin; rejects with
  `CheckoutSessionError` (`code`, `status`, `retryAfter`), e.g. `payment_in_progress`, `network_error`.
- `fallback: 'none'` → rejects `PopupBlockedError` instead. `redirectToCheckout(session)` skips the popup.
- `focusCheckout()` brings an open popup forward. A second `openCheckout()` supersedes the first
  (`closed`, reason `superseded`).
- If the page sends `Cross-Origin-Opener-Policy`, use `same-origin-allow-popups` (with `same-origin`
  the result is `closed`/`unreachable` while the payer may still be paying).

Result `status`: `succeeded` (payer's tx **confirmed** — not proof of payment), `canceled`,
`expired`, `closed` (**unknown**; `reason`: `closed_by_payer`, `unreachable`, `superseded`,
`returned_from_redirect`).

Recommended payer copy:

```ts
const MESSAGES = {
  succeeded: 'Thanks! We are confirming your payment.',
  canceled: 'Checkout canceled.',
  expired: 'That checkout link expired. Check your order status before trying again.',
  closed: 'We could not tell what happened. Check your order status before paying again.',
};
```

`<fianto-button>`: with a bundler, `import '@fianto/js/button'`; without one, serve
`node_modules/@fianto/js/dist/fianto-button.global.iife.js` from your own site (from a CDN: pin the
exact version + SRI `integrity` + `crossorigin="anonymous"`).

```html
<script src="/fianto-button.js"></script>
<fianto-button session-endpoint="/api/checkout" data-item="beans" label="buy"></fianto-button>
<script>
  document.querySelector('fianto-button').addEventListener('fianto:result', (e) => { /* e.detail.status */ });
  document.querySelector('fianto-button').addEventListener('fianto:error', (e) => console.error(e.detail.code));
</script>
```

`data-*` attributes become the JSON body (`data-item="beans"` → `{ "item": "beans" }`). Attributes:
`theme` (`brand`|`dark`|`light`|`outline`|`auto`), `label` (`plain`|`pay`|`buy`|`checkout`|`subscribe`|`donate`),
`shape` (`rect`|`rounded`|`pill`), `size` (`static`|`fill`), `locale` (`en`|`vi`), `fallback`, `disabled`.
CSS vars: `--fianto-button-height` (40–55px), `--fianto-button-radius`, `--fianto-button-width`,
`--fianto-button-focus-ring`.

## React: `@fianto/react`

```tsx
'use client';
import { FiantoButton, fetchCheckoutSession } from '@fianto/react';

export function BuyButton({ orderId }: { orderId: string }) {
  return (
    <FiantoButton
      session={() => fetchCheckoutSession('/api/checkout', { body: { orderId } })}
      label="pay"
      onResult={(result) => {/* map result.status to a message */}}
      onError={(error) => console.error(error)}
    />
  );
}
```

Props: `session` (required), `theme`, `label`, `shape`, `size`, `locale`, `fallback`, `loading`,
`disabled`, `onResult`, `onError`, `className`, `style`; `ref` → the `<button>`.

`useCheckout({ session, fallback? })` → `{ open, focus, status, result, error, isOpen }`. Call `open()`
directly in `onClick` (never after an `await` or in `useEffect`). `open()` never throws: it sets
`error` and resolves `undefined`.

No provider, no key: the server creates the session.
