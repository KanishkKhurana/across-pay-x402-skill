# Polyfilling `globalThis.fetch`

By default, `createX402WithAcross` returns a payment-wrapped `fetch` that you must call explicitly. If you'd rather have any library code that uses bare `fetch()` automatically negotiate x402 + bridging, opt into the polyfill.

## How

```js
import { createX402WithAcross } from 'across-pay-x402';

createX402WithAcross({
  account,
  polyfill: true,    // replaces globalThis.fetch
});

// Now any code path that uses fetch is x402-aware:
await fetch('https://seller.example/protected');
```

## When to use this

Use the polyfill when:

- You're integrating into a framework or SDK that calls `fetch()` internally and you can't pass it a custom fetch.
- You want minimal touch to the rest of the codebase.
- You're running a script that should "just work" with any URL.

## When NOT to use this

Don't use the polyfill when:

- You're running inside a long-lived server that also makes unrelated outbound HTTP — every call will try to negotiate 402, which means every non-402 response works fine but every 402 response triggers a payment.
- You have multiple buyer wallets in the same process — the polyfill is a single global; the last `createX402WithAcross({ polyfill: true })` wins.
- You want paid and unpaid call sites to be visibly distinct in code.

In those cases, just use the returned `fetch`:

```js
const { fetch: paidFetch } = createX402WithAcross({ account });
await paidFetch('https://seller.example/protected');
```

## Restoring or bypassing the original fetch

`createX402WithAcross` exposes `rawFetch`, which is the original `globalThis.fetch` captured **before** polyfilling:

```js
const { rawFetch } = createX402WithAcross({ account, polyfill: true });

// Later, for a request you don't want to pay for:
await rawFetch('https://free-api.example/whatever');
```

You can also restore the global if needed:

```js
globalThis.fetch = rawFetch;
```

## What gets replaced

`createX402WithAcross` calls `globalThis.fetch = paidFetch` exactly once when you set `polyfill: true`. It does **not**:

- Patch `node:fetch` or `undici` imports.
- Patch `XMLHttpRequest` or other HTTP APIs.
- Patch any `fetch` reference that was captured before the call.

If a library captured a reference like `const f = globalThis.fetch` before you ran `createX402WithAcross`, that library will keep using the unwrapped fetch. Order your initialization accordingly.
