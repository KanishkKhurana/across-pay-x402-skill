# Getting Started

This guide walks you through making your first crosschain x402-paid HTTP request from scratch.

## Step 1 — Install

```bash
npm init -y
npm install across-pay-x402
```

If you'll use the CDP wallet helper, `@coinbase/cdp-sdk` is a peer dependency declared by `across-pay-x402`. With npm 7+ it installs transitively; if you see "missing peer dependency" warnings, install it explicitly:

```bash
npm install @coinbase/cdp-sdk
```

If you'll use a local private-key wallet, install `viem` (also a peer dependency):

```bash
npm install viem
```

## Step 2 — Choose a wallet

| Wallet type | When to use | What you provide |
|---|---|---|
| Coinbase CDP server wallet | You don't want to manage keys yourself | `CDP_API_KEY_ID`, `CDP_API_KEY_SECRET`, `CDP_WALLET_SECRET` |
| Local private-key wallet | You manage your own key | `BUYER_PK` env var with a `0x`-prefixed private key |

See `guides/wallet-options.md` for full details on each.

## Step 3 — Fund the wallet

The buyer needs the input token (typically USDC) on **at least one** of the default origin chains. If you don't fund it, the SDK will throw:

```
No Across origin route with sufficient funds found for eip155:<X> <token>
```

The default origin chains the SDK scans:

- **Arbitrum One** (42161) — usually cheapest
- **Optimism** (10)
- **Polygon** (137)
- **Ethereum Mainnet** (1) — most expensive gas
- **Base** (8453)

Send a few cents of USDC to the buyer address on whichever you prefer (Arbitrum recommended for low fees).

## Step 4 — Write the script

Create `index.mjs`:

```js
import { createX402WithAcross, resolveCoinbaseWallet } from 'across-pay-x402';

const { account } = await resolveCoinbaseWallet();
console.log('Buyer address:', account.address);

const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({
  account,
  verbose: true,
});

const res = await paidFetch('https://your-seller.example/protected');
console.log('Status:', res.status);
console.log('Body:', await res.json());
console.log('bridgeInfo:', bridgeInfo);
```

## Step 5 — Run

```bash
export CDP_API_KEY_ID=...
export CDP_API_KEY_SECRET=...
export CDP_WALLET_SECRET=...
node index.mjs
```

## What to expect

### When destination already has funds

- `bridgeInfo.used === false`.
- The request completes quickly. x402 attaches a payment, seller settles.
- Total time: typically <2 seconds.

### When bridging is needed

- Verbose logging shows route discovery, parallel balance scan, quote selection.
- The bridge takes ~10–60 seconds depending on Across fill time.
- `bridgeInfo` is populated with both `depositTxHash` (origin) and `fillTxHash` (destination).
- After fill, x402 builds the payment and the seller settles normally.

## Next steps

- **`guides/tuning-bridge-behavior.md`** — speed up subsequent calls by pinning a chain
- **`guides/polyfilling-fetch.md`** — make all `fetch()` calls automatically pay
- **`examples/runnable-demo.md`** — a richer reference script with env-var knobs
- **`troubleshooting.md`** — if anything threw an error
