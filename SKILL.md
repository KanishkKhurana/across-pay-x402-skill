---
name: across-pay-x402
description: Build x402 micropayment buyers that auto-bridge funds across EVM chains via Across Protocol before payment. Use whenever an HTTP endpoint returns 402 Payment Required and the buyer may not already hold funds on the required destination chain (typically Base, eip155:8453). Wraps fetch with payment and crosschain pre-funding so consumers do not have to manage bridges manually.
---

# across-pay-x402

A buyer-side SDK that glues **Across Protocol** crosschain bridging onto the **x402** payment protocol. The buyer calls any x402-gated HTTP endpoint on one EVM chain (e.g. Base) using funds it already holds on a different EVM chain. The x402 SDK still owns protocol parsing, payload creation, retry, verification, and settlement — Across is invoked **only** to pre-fund the destination chain before x402 builds its payment payload.

## When to use this skill

Use this skill when the user wants to:
- Call x402-gated HTTP endpoints from a programmatic buyer.
- Pay on Base (or another EVM chain) using funds held on a *different* EVM chain.
- Avoid hand-rolling bridge transactions before each payment.
- Build an AI agent that autonomously consumes paid APIs across chains.

Do **NOT** use this skill when:
- The endpoint is not x402-gated (use whatever paywall protocol it actually uses).
- The buyer is not on an EVM chain (Solana, Bitcoin, etc.).
- You only need to bridge with no payment — use Across Protocol directly.
- You need *seller-side* x402 — this package is buyer-side only.

## What this skill enables vs. what the user must provide

| The agent can ... | The user / caller must provide ... |
|---|---|
| Create or retrieve a CDP-managed buyer wallet | `CDP_API_KEY_ID`, `CDP_API_KEY_SECRET`, `CDP_WALLET_SECRET` env vars |
| Use a local private-key wallet instead of CDP | A `0x`-prefixed private key, never hard-coded |
| Auto-discover which origin chain to bridge from | The buyer wallet must hold the input token (typically USDC) on at least one supported origin chain |
| Pre-fund the destination chain before each x402 payment | (handled automatically) |
| Call any x402-gated URL with a wrapped fetch | The endpoint URL |
| Polyfill `globalThis.fetch` so 3rd-party libraries auto-pay | (optional opt-in via `polyfill: true`) |
| Inspect what happened on each call (chain, fees, tx hashes) | (read `bridgeInfo` after each paid call) |
| Force the bridge path for testing | Set `across.forceBridge: true` |
| Override RPCs for production-grade endpoints | A map of `{ chainId: rpcUrl }` |

## Prerequisites

- **Node.js 20+** (the package ships ESM only).
- `npm install across-pay-x402` (or pnpm/yarn equivalent).
- A buyer wallet **funded** with the required input token (typically USDC) on at least one of the default origin chains: **Arbitrum, Optimism, Polygon, Ethereum, Base**.
- For the CDP wallet helper (`resolveCoinbaseWallet`): a Coinbase Developer Platform account and credentials. See `guides/wallet-options.md`.

## Quickstart

Minimum code to make a paid request:

```js
import { createX402WithAcross, resolveCoinbaseWallet } from 'across-pay-x402';

// 1. Get a buyer wallet (CDP-managed; private key never leaves Coinbase).
const { account } = await resolveCoinbaseWallet();

// 2. Build the payment-wrapped fetch.
const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({
  account,
  verbose: true,
});

// 3. Call any x402-gated endpoint. Bridging happens automatically if needed.
const res = await paidFetch('https://your-x402-seller.example/protected');
console.log(await res.json());
console.log('bridgeInfo:', bridgeInfo);
```

Required environment variables for `resolveCoinbaseWallet()`:

```bash
export CDP_API_KEY_ID=...
export CDP_API_KEY_SECRET=...
export CDP_WALLET_SECRET=...
```

Then: `node yourscript.mjs`.

## Core API at a glance

The package exports exactly two functions and several types:

### `createX402WithAcross(config)`

Returns `{ account, bridgeInfo, client, httpClient, fetch, rawFetch, rpcs }`. The `fetch` field is the wrapped fetch you should call — every paid request will trigger the bridging hook automatically.

### `resolveCoinbaseWallet()`

Returns `{ provider, source, account }` where `account` is a viem `Account` backed by a Coinbase CDP server wallet. Never exposes the private key.

Full reference (every type, every field, every default): **`reference/api.md`**.

## Decision tree

```
Does the buyer want managed key custody?
├─ Yes → use resolveCoinbaseWallet()             (requires CDP credentials)
└─ No  → use viem's privateKeyToAccount(0x...)   (requires the buyer's PK in env)

Does the buyer reliably have funds on the same origin chain?
├─ Yes → set across.preferredOriginChainId (fast path)
└─ No  → leave it unset; SDK scans all default origins in parallel

Are you testing the crosschain path?
├─ Yes → set across.forceBridge = true            (bridges even if destination is funded)
└─ No  → leave it; bridging only fires on shortfall

Are you running in production?
├─ Yes → set across.rpcs with paid RPC endpoints  (defaults are public + rate-limited)
└─ No  → defaults are fine for local dev
```

## How it works (high level)

1. The caller invokes the wrapped `fetch`.
2. The x402 SDK hits the URL, gets a `402 Payment Required` with payment requirements.
3. **Before** x402 constructs the payment payload, this SDK's `onBeforePaymentCreation` hook fires:
   - Reads the buyer's balance of the required asset on the destination chain.
   - If sufficient (and `forceBridge` is false), skip.
   - Otherwise compute `shortfall = amount - balance + gasBuffer`, pick an origin chain that has funds, fetch an Across quote with `tradeType=exactOutput`, send the approval + swap transactions via viem, then poll Across `/deposit/status` until `filled`.
4. x402 builds the payment payload using the (now-funded) destination balance.
5. x402 replays the original request with an `X-PAYMENT` header; seller verifies and settles.

Detailed step-by-step flow: **`reference/flow.md`**.

## Where to read next

- **`reference/api.md`** — every function, type, and return shape
- **`reference/configuration.md`** — every config option, default, and env var
- **`reference/flow.md`** — internal bridging flow, in detail
- **`reference/across-protocol.md`** — the three Across HTTP endpoints used
- **`reference/supported-chains.md`** — chain table with default RPCs
- **`guides/getting-started.md`** — step-by-step tutorial from `npm install` to first paid call
- **`guides/wallet-options.md`** — CDP vs. local PK wallet
- **`guides/polyfilling-fetch.md`** — replace `globalThis.fetch`
- **`guides/tuning-bridge-behavior.md`** — gasBuffer, forceBridge, preferred chain, custom RPCs
- **`examples/code-snippets.md`** — drop-in patterns for common cases
- **`examples/runnable-demo.md`** — a complete runnable demo with env-var knobs
- **`troubleshooting.md`** — common errors and fixes

## Invariants worth remembering

- Mainnet target is **Base (`eip155:8453`)**.
- Default origin chains scanned: `[42161, 10, 137, 1, 8453]` (Arbitrum, Optimism, Polygon, Ethereum, Base).
- `bridgeInfo` is a **single mutable object**, reset and repopulated on every paid call. Snapshot it (`{ ...bridgeInfo }`) before the next call if you need to retain it.
- Public default RPCs will rate-limit you in production. Override via `across.rpcs`.
- Across fill polling: 2 s interval, 120 s timeout (not currently configurable from the public API).
- The SDK never reads private keys, never logs them, and never exposes them through `bridgeInfo`.
