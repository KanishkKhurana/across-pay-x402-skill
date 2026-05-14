# API Reference

The `across-pay-x402` package exports exactly two functions and several types. Everything below is verbatim against `across-pay-x402@1.2.0`.

## All exports

```ts
import {
  createX402WithAcross,
  resolveCoinbaseWallet,
  // deprecated alias for resolveCoinbaseWallet:
  resolveCoinbaseDemoWallet,
  // types:
  type AcrossConfig,
  type BridgeInfo,
  type CoinbaseWallet,
  type CoinbaseDemoWallet, // deprecated alias for CoinbaseWallet
} from 'across-pay-x402';
```

---

## `createX402WithAcross(config)`

The main entry. Constructs an x402 client with an Across pre-funding hook attached, and returns a payment-wrapped fetch.

### Signature

```ts
function createX402WithAcross(config: {
  account: Account;            // viem Account (any signer)
  across?: AcrossConfig;       // crosschain bridging options
  polyfill?: boolean;          // if true, replaces globalThis.fetch
  verbose?: boolean;           // if true, logs decisions to stdout
}): {
  account: Account;
  bridgeInfo: BridgeInfo;      // mutable; reset and repopulated per paid request
  client: x402Client;          // raw x402 client (Across hook already registered)
  httpClient: x402HTTPClient;  // x402 HTTP client wrapping `client`
  fetch: (input: RequestInfo | URL, init?: RequestInit) => Promise<Response>;
  rawFetch: typeof fetch;      // pre-wrap globalThis.fetch
  rpcs: Record<number, string>;// merged RPC map (DEFAULT_RPCS + your overrides)
};
```

### Parameter details

- **`account`** — required. Any viem `Account`. Examples: from `resolveCoinbaseWallet()`, or `privateKeyToAccount(...)` from `viem/accounts`, or any other viem-compatible signer.
- **`across`** — optional. See `AcrossConfig` below and `reference/configuration.md` for the full table.
- **`polyfill`** — optional, default `false`. When `true`, sets `globalThis.fetch = paidFetch` so any library code that uses bare `fetch()` will transparently negotiate x402 + bridging.
- **`verbose`** — optional, default `false`. When `true`, prints `[x402+across]` log lines describing each decision (route discovery, balance scan, quote selection, bridge txs).

### Return details

- **`fetch`** — the payment-wrapped fetch. Call this in place of `globalThis.fetch`. On a 402, the x402 SDK triggers the bridging hook (if needed) and replays with payment.
- **`rawFetch`** — the original `globalThis.fetch` captured at construction time. Use this for requests you don't want to pay for, especially when `polyfill: true`.
- **`bridgeInfo`** — a **live mutable object**. Reset and repopulated by the hook on every paid request. Read it immediately after the `await paidFetch(...)` call you care about, or snapshot with `{ ...bridgeInfo }`.
- **`client`**, **`httpClient`** — exposed for advanced users who want to call lower-level x402 APIs.
- **`rpcs`** — the merged RPC map (defaults + your overrides), useful if you want to reuse the same RPCs elsewhere in your code.

### Example

```js
import { createX402WithAcross } from 'across-pay-x402';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount(process.env.BUYER_PK);

const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({
  account,
  verbose: true,
  across: {
    preferredOriginChainId: 42161,
    gasBuffer: 5_000n,
  },
});

const res = await paidFetch('https://seller.example/protected');
```

---

## `resolveCoinbaseWallet()`

Creates or retrieves a Coinbase CDP server-managed wallet. The private key never leaves Coinbase infrastructure; signing happens via the CDP API.

### Signature

```ts
function resolveCoinbaseWallet(): Promise<{
  provider: 'coinbase-cdp';
  source: 'CDP server wallet';
  account: Account;   // viem Account
}>;
```

### Required environment variables

Read by the underlying `@coinbase/cdp-sdk`:

- `CDP_API_KEY_ID`
- `CDP_API_KEY_SECRET`
- `CDP_WALLET_SECRET`

### Behavior

Calls `cdp.evm.getOrCreateAccount({ name: 'x402-buyer' })`. The wallet is identified by the fixed name `"x402-buyer"`, so repeated calls return the **same** wallet (idempotent). The address is stable — fund once and reuse.

### Example

```js
import { resolveCoinbaseWallet } from 'across-pay-x402';

const { account, provider, source } = await resolveCoinbaseWallet();
console.log(`[${provider}/${source}] Buyer: ${account.address}`);
```

---

## Types

### `AcrossConfig`

```ts
interface AcrossConfig {
  preferredOriginChainId?: number;     // try this origin first (fast path)
  preferredOriginChainOnly?: boolean;  // if true, fail when preferred is unfunded
  originChainIds?: number[];           // whitelist of origins to consider
  gasBuffer?: bigint;                  // extra units added to shortfall
  forceBridge?: boolean;               // always bridge, even if destination is funded
  rpcs?: Record<number, string>;       // per-chain RPC overrides
}
```

See `reference/configuration.md` for the full table of defaults and behavior.

### `BridgeInfo`

```ts
interface BridgeInfo {
  used: boolean;                       // false if no bridge happened
  forced?: boolean;                    // true if forceBridge was set
  originChainId?: number;
  destinationChainId?: number;
  inputToken?: string;                 // ERC-20 contract address on origin
  outputToken?: string;                // ERC-20 contract address on destination
  inputAmount?: string;                // integer string, raw token units
  destinationAmount?: string;          // integer string, raw token units
  depositTxHash?: `0x${string}`;       // origin tx hash
  fillTxHash?: string;                 // destination tx hash
}
```

Repopulated on every paid request. On every paid request the hook resets *all* fields, then writes:

- Always: `used` (initially `false`), `forced`, `destinationChainId`, `outputToken`, `destinationAmount`.
- Only when a bridge actually executes: `used = true`, `originChainId`, `inputToken`, `inputAmount`, `depositTxHash`, `fillTxHash`.

If `used === false`, the destination already had enough funds (or you didn't go through a paid call yet).

### `CoinbaseWallet`

```ts
interface CoinbaseWallet {
  provider: 'coinbase-cdp';
  source: 'CDP server wallet';
  account: ReturnType<typeof toAccount>;   // viem Account
}
```

---

## Deprecated aliases

Kept for backwards compatibility; do not use in new code:

- `resolveCoinbaseDemoWallet` → use `resolveCoinbaseWallet`
- `CoinbaseDemoWallet` (type) → use `CoinbaseWallet`

---

## Peer dependencies

The package declares the following peer dependencies. `viem` is always required; `@coinbase/cdp-sdk` is only required if you use `resolveCoinbaseWallet`.

- `@coinbase/cdp-sdk` `>=1.0.0` (peer, only for the CDP wallet helper)
- `viem` `>=2.0.0` (peer, always required)
