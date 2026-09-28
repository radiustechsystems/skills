# Accept x402 Payments with `radius-sdk`

Use `radius-sdk/hono` for Hono applications and Cloudflare Workers. It creates
x402 challenges, verifies and settles paid requests through the configured
facilitator, and exposes the payment receipt to handlers. Do not hand-roll the
challenge or `/verify` and `/settle` flow for ordinary integrations.

## Install

```bash
pnpm add radius-sdk hono
```

The root and Hono entry points do not load viem at runtime. The middleware does
no I/O at module scope, so it is suitable for Cloudflare Workers.

## Protect routes

```typescript
import { Hono } from 'hono';
import { radiusPayments, type RadiusPaymentVariables } from 'radius-sdk/hono';

type Env = {
  Bindings: { PAY_TO: `0x${string}` };
  Variables: RadiusPaymentVariables;
};

const app = new Hono<Env>();

app.use(
  '/api/*',
  radiusPayments<Env>({
    network: 'testnet',
    payTo: (c) => c.env.PAY_TO,
    routes: {
      'GET /api/lookup': { price: '0.001 SBC', description: 'One lookup' },
      'POST /api/query': '0.01 SBC',
    },
  }),
);

app.get('/api/lookup', (c) => c.json({ result: 'ok' }));
app.post('/api/query', (c) => c.json({ results: [] }));

export default app;
```

The server needs a recipient address, not a signing key. Route keys are
`METHOD /path` with Hono-style `*` wildcards. Requests that match no configured
route pass through without payment.

## Options

| Option | Default | Purpose |
|---|---|---|
| `payTo` | Required | Address or context function that selects the recipient |
| `routes` | Required | Route map; a bare price is shorthand for `{ price }` |
| `network` | `mainnet` | `mainnet`, `testnet`, or an explicit `RadiusNetwork` |
| `settle` | `before` | Settle before the handler, or `after` a response below 400 |
| `gasSponsoring` | `auto` | Advertise the facilitator's EIP-2612 gas sponsorship support |
| `facilitator` | Radius facilitator | Hosted options or a `FacilitatorClient` |
| `onSettled` | None | Observe every settled payment |

A route may define `price`, `payTo`, `description`, `mimeType`, and
`maxTimeoutSeconds`. Prices may be display-unit strings such as `0.001 SBC` or
base-unit objects such as `{ amount: '1000' }`; SBC has six decimals. A price or
recipient may be a function of the Hono context.

## Read receipts

```typescript
app.get('/api/lookup', (c) => {
  const receipt = c.get('radiusPayment');
  return c.json({
    result: 'ok',
    transaction: receipt?.transaction,
    payer: receipt?.payer,
  });
});
```

The receipt includes `success`, `transaction`, `payer`, `amount`, `network`,
`explorerUrl`, and failure fields when applicable. The response also carries a
`PAYMENT-RESPONSE` header for the buyer.

Use `onSettled` for centralized logging:

```typescript
radiusPayments<Env>({
  network: 'testnet',
  payTo: (c) => c.env.PAY_TO,
  routes: { 'GET /api/lookup': '0.001 SBC' },
  onSettled: (receipt, c) =>
    console.log('paid', c.req.path, receipt.payer, receipt.transaction),
});
```

## Settlement timing

`settle: 'before'` is the SDK default. Settlement completes before the handler
runs, so protected work is not performed for an unsettled payment. If the
handler later fails, the payer may have paid without receiving the resource.
Log the transaction hash with delivery failures and implement an idempotent
retry or refund policy.

`settle: 'after'` verifies first, runs the handler, and settles only when the
handler returns a status below 400. This avoids charging for error responses but
performs the handler work before payment is final.

## Facilitators and networks

Mainnet is the SDK default. Use `network: 'testnet'` during Testnet development.
The network choice switches the Radius facilitator, chain, and default asset;
it does not switch your recipient address. Supply the correct recipient for the
selected network.

For another hosted facilitator, pass `facilitator: { url, apiKey }`. The API key
is sent as `x-api-key`. Advanced integrations may pass a `FacilitatorClient`
from `@x402/core/server`. Query `/supported` before depending on a facilitator's
network, transfer method, or gas-sponsoring extensions.

## Validation checklist

1. Confirm an unpaid protected route returns 402 and `PAYMENT-REQUIRED`.
2. Confirm an unconfigured route remains free.
3. Pay on Testnet from a different funded wallet.
4. Confirm handler receipt fields and `PAYMENT-RESPONSE`.
5. Confirm the settlement transaction on the selected network.
6. Exercise handler failure under the chosen settlement mode.

Use the low-level [facilitator API](facilitator-api.md) only for custom protocol
work that the middleware cannot express.
