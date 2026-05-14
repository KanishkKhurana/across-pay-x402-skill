# Runnable Demo

A complete, copy-pasteable demo script with env-var knobs for tweaking behavior.

## File

Save as `demo.mjs` in a project that has `across-pay-x402` installed (`npm install across-pay-x402`):

```js
// Runnable x402 + Across demo using a Coinbase CDP server wallet.
//
// Required env:
//   CDP_API_KEY_ID
//   CDP_API_KEY_SECRET
//   CDP_WALLET_SECRET
//
// Optional env:
//   X402_URL            Target endpoint (default: CoinGecko x402 simple price)
//   PREFERRED_CHAIN_ID  Preferred Across origin chain id (e.g. 42161)
//   PREFERRED_ONLY      "1" to fail if the preferred chain is unfunded
//   FORCE_BRIDGE        "1" to always bridge even if destination is funded
//   GAS_BUFFER          Extra units added to the shortfall (default: 1000)
//
// Run:
//   node demo.mjs

import { createX402WithAcross, resolveCoinbaseWallet } from 'across-pay-x402';

const DEFAULT_URL =
  'https://pro-api.coingecko.com/api/v3/x402/simple/price?vs_currencies=usd&symbols=btc,eth,sol';

function requireEnv(name) {
  if (!process.env[name]) {
    console.error(`Missing required env: ${name}`);
    process.exit(1);
  }
}

requireEnv('CDP_API_KEY_ID');
requireEnv('CDP_API_KEY_SECRET');
requireEnv('CDP_WALLET_SECRET');

const url = process.env.X402_URL ?? DEFAULT_URL;
const preferredOriginChainId = process.env.PREFERRED_CHAIN_ID
  ? Number(process.env.PREFERRED_CHAIN_ID)
  : undefined;
const preferredOriginChainOnly = process.env.PREFERRED_ONLY === '1';
const forceBridge = process.env.FORCE_BRIDGE === '1';
const gasBuffer = process.env.GAS_BUFFER ? BigInt(process.env.GAS_BUFFER) : undefined;

console.log('[demo] Resolving Coinbase CDP wallet...');
const { account } = await resolveCoinbaseWallet();
console.log(`[demo] Buyer address: ${account.address}`);

const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({
  account,
  verbose: true,
  across: {
    preferredOriginChainId,
    preferredOriginChainOnly,
    forceBridge,
    gasBuffer,
  },
});

console.log(`[demo] Calling x402 endpoint: ${url}`);
const started = Date.now();
const res = await paidFetch(url);
const elapsedMs = Date.now() - started;
console.log(`[demo] Response ${res.status} ${res.statusText} in ${elapsedMs}ms`);

const contentType = res.headers.get('content-type') ?? '';
const body = contentType.includes('application/json') ? await res.json() : await res.text();
console.log('[demo] Body:', body);

console.log('[demo] Bridge info:');
console.log(JSON.stringify(bridgeInfo, null, 2));

if (bridgeInfo.used) {
  console.log(
    `[demo] Bridged from chain ${bridgeInfo.originChainId} → ${bridgeInfo.destinationChainId}`,
  );
  console.log(`[demo]   deposit tx: ${bridgeInfo.depositTxHash}`);
  console.log(`[demo]   fill tx:    ${bridgeInfo.fillTxHash}`);
} else {
  console.log('[demo] No bridge was needed (destination already funded).');
}
```

## Run it

### Basic (auto-detect origin chain)

```bash
export CDP_API_KEY_ID=...
export CDP_API_KEY_SECRET=...
export CDP_WALLET_SECRET=...
node demo.mjs
```

### Custom seller URL

```bash
X402_URL=https://your-seller.example/protected node demo.mjs
```

### Fast path on Arbitrum

```bash
PREFERRED_CHAIN_ID=42161 node demo.mjs
```

### Strict single-origin (Arbitrum only)

```bash
PREFERRED_CHAIN_ID=42161 PREFERRED_ONLY=1 node demo.mjs
```

### Force the crosschain path

```bash
FORCE_BRIDGE=1 node demo.mjs
```

### Larger gas cushion (0.05 USDC instead of 0.001 USDC)

```bash
GAS_BUFFER=50000 node demo.mjs
```

## Expected output (truncated)

```
[demo] Resolving Coinbase CDP wallet...
[demo] Buyer address: 0xAbc...
[x402+across] Buyer: 0xAbc...
[x402+across] Dynamic chain discovery enabled
[demo] Calling x402 endpoint: https://...
[x402+across] Payment requirements:
  Network: eip155:8453
  Asset: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
  Amount: 10000
  Destination balance: 0
[x402+across] Bridge needed — shortfall: 11000 (includes gas buffer)
  [route discovery] Found 4 route(s) from 42161, 10, 137, 1
  [parallel scan] Done in 482ms with 1 funded chain(s)
  [route selection] Using only funded candidate on chain 42161
[across] Sending approval tx...
[x402+across] Bridging from chain 42161 to chain 8453
[x402+across] Bridge complete — fill tx: 0xDef...
[demo] Response 200 OK in 18402ms
[demo] Body: { ... }
[demo] Bridge info: { ... }
```

## Variant — local private-key wallet

Swap the wallet section for a viem `privateKeyToAccount`:

```js
import { createX402WithAcross } from 'across-pay-x402';
import { privateKeyToAccount } from 'viem/accounts';

requireEnv('BUYER_PK');
const account = privateKeyToAccount(process.env.BUYER_PK);
```

Run with:

```bash
export BUYER_PK=0x...
node demo.mjs
```

See `guides/wallet-options.md` for security considerations.
