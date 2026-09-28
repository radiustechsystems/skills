# Pay x402 Resources with `radius-sdk`

Use `radius-sdk/client` for TypeScript applications and `radius-cli wallet x402`
for one-off agent or terminal requests. These surfaces parse the challenge,
select a compatible offer, sign, pay, and retry the request.

## TypeScript client

```bash
pnpm add radius-sdk viem
```

```typescript
import { createRadiusFetch } from 'radius-sdk/client';

const payFetch = createRadiusFetch({
  network: 'testnet',
  signer: process.env.RADIUS_PRIVATE_KEY as `0x${string}`,
  maxPerRequest: '0.01 SBC',
});

const response = await payFetch('https://seller.example/api/lookup');
console.log(response.status, await response.text());
```

The returned function accepts normal `fetch` arguments. Non-402 responses are
returned unchanged. A signer may be a secret-manager-provided private key, a
viem account, or a viem `WalletClient`. Never log or commit key material.

## Payment policy

`maxPerRequest` is mandatory. It caps one payment and is not a cumulative
budget. The client also pays only the configured network and asset and takes
the first compatible offer within the cap, in server order.

Use `onPaymentRequired` for a recipient allowlist or total budget:

```typescript
const budget = 1_000_000n; // 1 SBC in six-decimal base units
let committed = 0n;

const payFetch = createRadiusFetch({
  network: 'testnet',
  signer: process.env.RADIUS_PRIVATE_KEY as `0x${string}`,
  maxPerRequest: '0.01 SBC',
  onPaymentRequired: (offer) => {
    if (offer.payTo.toLowerCase() !== '0xselleraddress'.toLowerCase()) return false;
    if (committed + BigInt(offer.amount) > budget) return false;
    committed += BigInt(offer.amount);
    return true;
  },
});
```

An offer includes `amount`, `amountFormatted`, `payTo`, `asset`, `network`,
`resource`, `scheme`, `x402Version`, `transferMethod`, `gasSponsored`, and the
raw requirements.

## Receipts and reconciliation

```typescript
import { getPaymentReceipt } from 'radius-sdk/client';

const response = await payFetch('https://seller.example/api/lookup');
const receipt = getPaymentReceipt(response, payFetch.network);
if (receipt) console.log(receipt.transaction, receipt.amount, receipt.explorerUrl);
```

For a timeout or ambiguous delivery, use `payFetch.getSettlement(txHash)` before
paying again. It returns `undefined` while the node does not know the transaction
and otherwise reports status, transfers, recipient totals, and explorer URL.

## Errors

```typescript
import { RadiusPaymentError } from 'radius-sdk/client';

try {
  await payFetch('https://seller.example/api/lookup');
} catch (error) {
  if (!(error instanceof RadiusPaymentError)) throw error;
  console.error(error.code, error.message);
}
```

Handle at least these codes: `price_above_limit`, `declined`,
`network_mismatch`, `asset_mismatch`, `no_compatible_offer`,
`unsupported_transfer_method`, `invalid_challenge`, `payment_rejected`,
`invalid_receipt`, `redirect_refused`, `approval_required`, and
`approval_failed`.

The paid retry never follows a redirect to another origin. For
`payment_rejected`, `error.details.response` contains the server response.

## Permit2 approval

The Radius facilitator sponsors the one-time Permit2 approval through an
EIP-2612 signature, so a payer holding only SBC can pay without sending an
approval transaction. For a facilitator without sponsorship,
`permit2Approval: 'auto'` sends one unlimited Permit2 approval before the first
payment. Use `permit2Approval: 'never'` or `onApprovalRequired` when the caller
must explicitly control that transaction.

## Agent and terminal client

Use `radius-cli` 0.2.0 or later:

```bash
radius-cli --version

RADIUS_HOME=.radius RADIUS_NETWORK=testnet \
  radius-cli wallet x402 get https://seller.example/api/lookup \
  --x402-threshold 0.01 \
  --json \
  -y
```

`--x402-threshold` is a display-unit hard cap for one request. When combined
with `-y`, an offer above the cap is refused with exit code 2. `-y` without a
threshold authorizes any amount and is unsuitable for unattended agents.

POST example:

```bash
RADIUS_HOME=.radius RADIUS_NETWORK=testnet \
  radius-cli wallet x402 post https://seller.example/api/query \
  -H "Content-Type: application/json" \
  -d '{"query":"radius"}' \
  --x402-threshold 0.01 \
  --json \
  -y
```

Supported verbs are `get`, `post`, `put`, `patch`, `delete`, `head`, and
`options`. `-d` accepts a literal, `@path`, or stdin via `-`.

The low-level typed-data templates, [curl/cast fallback](x402-cli-cast.md), and
`scripts/x402-pay.mjs` are for custom or legacy environments. Do not use them as
the default application or agent workflow.
