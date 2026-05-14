# Code Snippets

Drop-in patterns for common situations. All snippets assume `import { createX402WithAcross, resolveCoinbaseWallet } from 'across-pay-x402';` at the top of the file.

## Minimal (CDP wallet)

```js
const { account } = await resolveCoinbaseWallet();

const { fetch: paidFetch } = createX402WithAcross({ account });

const res = await paidFetch('https://seller.example/protected');
console.log(await res.json());
```

## Minimal (local PK)

```js
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount(process.env.BUYER_PK);

const { fetch: paidFetch } = createX402WithAcross({ account });

const res = await paidFetch('https://seller.example/protected');
console.log(await res.json());
```

## Pinned origin chain (fast path)

```js
const { fetch: paidFetch } = createX402WithAcross({
  account,
  across: { preferredOriginChainId: 42161 },   // Arbitrum
});
```

## Strict single-origin (no fallback)

```js
const { fetch: paidFetch } = createX402WithAcross({
  account,
  across: {
    preferredOriginChainId: 42161,
    preferredOriginChainOnly: true,
  },
});
```

## Custom RPCs

```js
const { fetch: paidFetch } = createX402WithAcross({
  account,
  across: {
    rpcs: {
      8453:  process.env.BASE_RPC,
      42161: process.env.ARB_RPC,
    },
  },
});
```

## Force-bridge for testing

```js
const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({
  account,
  verbose: true,
  across: { forceBridge: true },
});

await paidFetch('https://seller.example/protected');
console.assert(bridgeInfo.used && bridgeInfo.forced, 'expected forced bridge');
```

## Polyfill global fetch

```js
createX402WithAcross({ account, polyfill: true });

// Any subsequent call works transparently:
await fetch('https://seller.example/protected');
```

## Inspecting the bridge result

```js
const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({ account });

await paidFetch('https://seller.example/protected');

if (bridgeInfo.used) {
  console.log({
    from:               bridgeInfo.originChainId,
    to:                 bridgeInfo.destinationChainId,
    inputToken:         bridgeInfo.inputToken,
    outputToken:        bridgeInfo.outputToken,
    inputAmount:        bridgeInfo.inputAmount,
    destinationAmount:  bridgeInfo.destinationAmount,
    deposit:            bridgeInfo.depositTxHash,
    fill:               bridgeInfo.fillTxHash,
  });
} else {
  console.log('Destination already funded — no bridge.');
}
```

## Reusing rawFetch for unpaid requests

```js
const {
  fetch: paidFetch,
  rawFetch,
} = createX402WithAcross({ account, polyfill: true });

// Paid request:
await paidFetch('https://seller.example/protected');
// or via the polyfilled global:
await fetch('https://seller.example/protected');

// Unpaid request (bypasses x402):
await rawFetch('https://public-api.example/anything');
```

## Awaiting requests sequentially (bridgeInfo is shared)

```js
const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({ account });

// Avoid Promise.all on paid requests — bridgeInfo is mutated per-request and will race.
await paidFetch('https://seller.example/a');
const aInfo = { ...bridgeInfo };

await paidFetch('https://seller.example/b');
const bInfo = { ...bridgeInfo };

console.log({ aInfo, bInfo });
```

## Multiple concurrent buyers (one client each)

```js
async function buyOnce(account, url) {
  const { fetch: paidFetch, bridgeInfo } = createX402WithAcross({ account });
  const res = await paidFetch(url);
  return { res, bridgeInfo: { ...bridgeInfo } };
}

const [a, b] = await Promise.all([
  buyOnce(accountA, 'https://seller.example/x'),
  buyOnce(accountB, 'https://seller.example/y'),
]);
```

## Larger gas buffer on destination

```js
const { fetch: paidFetch } = createX402WithAcross({
  account,
  across: { gasBuffer: 50_000n },   // 0.05 USDC, since USDC has 6 decimals
});
```

## Catching common errors

```js
try {
  await paidFetch('https://seller.example/protected');
} catch (err) {
  const msg = String(err);
  if (msg.includes('No Across origin route with sufficient funds')) {
    console.error('Buyer is unfunded on every candidate origin chain.');
  } else if (msg.includes('Across deposit expired')) {
    console.error('Across relayer did not fill in time; retry later.');
  } else if (msg.includes('Across fill timed out')) {
    console.error('Fill took longer than 120s; check Across status.');
  } else {
    throw err;
  }
}
```
