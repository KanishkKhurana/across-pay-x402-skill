# Troubleshooting

Common errors thrown by the SDK or by Across, mapped to root cause and fix.

## `Missing required env: CDP_API_KEY_ID` (and `_SECRET`, `_WALLET_SECRET`)

You called `resolveCoinbaseWallet()` without setting CDP env vars.

**Fix.** Set `CDP_API_KEY_ID`, `CDP_API_KEY_SECRET`, `CDP_WALLET_SECRET` in your environment. See `guides/wallet-options.md`.

---

## `Unsupported network format: <X>. Expected eip155:<chainId>`

The x402 seller returned a `network` field that doesn't match the `eip155:<chainId>` shape. This SDK is EVM-only.

**Fix.** Confirm the seller advertises an EVM chain. If you're hitting a non-EVM x402 endpoint, this SDK won't help.

---

## `No Across origin route with sufficient funds found for eip155:<X> <token>`

The buyer holds no balance of the required asset on any candidate origin chain, OR none of the candidate routes can satisfy the shortfall.

**Diagnoses:**

1. **Wallet unfunded.** Send the required asset (e.g. USDC) to the buyer address on one of the default origins (Arbitrum, Optimism, Polygon, Ethereum, Base).
2. **Funded on a non-default chain.** Add that chain to `across.originChainIds`.
3. **Balance below quote.** The quote's `inputAmount` exceeded your balance. Fund more, or call a smaller-amount endpoint.

Run with `verbose: true` and look at `[route discovery]` and `[parallel scan]` to see what the SDK saw.

---

## `Preferred origin chain <X> has no funded route`

You set `preferredOriginChainOnly: true` and the preferred chain has zero balance for the route.

**Fix.** Either fund the preferred chain, drop `preferredOriginChainOnly`, or change `preferredOriginChainId` to a funded one.

---

## `No Across route from preferred chain <X> to chain <Y>`

You set `preferredOriginChainOnly: true` and Across has no route at all from that origin to that destination for the requested token.

**Fix.** Pick a different origin, or drop `preferredOriginChainOnly` to let the SDK fall back to scanning.

---

## `Insufficient origin balance. Need <X> on chain <Y>, have <Z>`

A race between the balance read and the quote return: the quote came back asking for more than the balance (typically because of fee fluctuation between calls).

**Fix.** Fund the origin with more headroom. The default `gasBuffer` is `1000n`; for production raise it (e.g. `50_000n` for USDC).

---

## `Across quote failed (<status>): <body>`

Across's `/api/swap/approval` rejected the request. Common causes:

- Route doesn't exist for that pair.
- `amount` too small (below the route's minimum).
- API outage.

**Fix.** Read the error body. Try a different origin. Confirm the route at:

```
https://app.across.to/api/available-routes?destinationChainId=<id>
```

---

## `Across deposit expired without fill`

The Across relayer didn't fill within the protocol's window.

**Fix.** Retry. If repeated, check Across status via https://docs.across.to/ for relayer issues. Note: even after this throws, funds are NOT lost — Across will refund or eventually fill, see the Across docs for the recovery flow.

---

## `Across fill timed out after 120s`

The SDK's poll timeout fired but the deposit may still fill eventually on-chain.

**Fix.** The deposit tx hash is **not** exposed on the error itself; if you need it for follow-up, run with `verbose: true` so it appears in the logs. You can then manually poll:

```bash
curl "https://app.across.to/api/deposit/status?originChainId=<origin>&depositTxHash=0x..."
```

The 120 s timeout is hard-coded in the public API and not currently configurable.

---

## RPC rate limits

Symptoms: random `ContractFunctionExecutionError`, slow balance reads, missing fills, occasional non-deterministic failures.

**Fix.** Provide your own RPCs via `across.rpcs`. Free public RPCs are not reliable for production. See `reference/configuration.md`.

---

## "It says no bridge happened, but I expected one"

`bridgeInfo.used === false` means the destination already had enough.

**Fix.**

- If you wanted to test the bridge path, set `forceBridge: true`.
- If you didn't expect funds to be there, check the buyer address on the destination chain explorer.

---

## Concurrent calls clobber `bridgeInfo`

`bridgeInfo` is one mutable object. Two `paidFetch` calls in parallel will race; whichever finishes last wins.

**Fix.** Either `await` sequentially and snapshot `bridgeInfo` between calls (`{ ...bridgeInfo }`), or construct one `createX402WithAcross` client per concurrent caller.

---

## TypeScript: "Cannot find module 'across-pay-x402' or its corresponding type declarations"

The package ships `.d.ts` files in `dist/` and exports them via `package.json#exports`. You probably need a modern `moduleResolution`.

**Fix.** In `tsconfig.json`:

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}
```

Or `"moduleResolution": "NodeNext"` with `"module": "NodeNext"`.

---

## ESM: "ReferenceError: require is not defined" or "Cannot use import statement outside a module"

The package is ESM-only (`"type": "module"` in its `package.json`). Your consumer must also be ESM.

**Fix.** Either:

- Use a `.mjs` extension for your entry file, OR
- Set `"type": "module"` in your project's `package.json`, OR
- Compile via a bundler that emits ESM.

---

## Verifying the wallet is funded

Quick sanity check (no SDK needed):

```bash
# USDC on Arbitrum One — replace with your buyer address
curl -s -X POST https://arb1.arbitrum.io/rpc \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_call","params":[{"to":"0xaf88d065e77c8cC2239327C5EDb3A432268e5831","data":"0x70a08231000000000000000000000000<YOUR_BUYER_ADDRESS_WITHOUT_0x>"},"latest"],"id":1}'
```

A non-zero `result` (e.g. `0x...1e8480`) means the wallet has USDC. `0x0` means empty.
