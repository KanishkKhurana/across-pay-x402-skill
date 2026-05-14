# Bridging Flow

This document describes exactly what happens between calling `paidFetch(url)` and the seller settling the payment. All steps are derived from the source of `across-pay-x402@1.2.0`.

## High-level sequence

```
caller       paidFetch       x402 SDK            Across hook         Across API       chain
  │              │                │                    │                   │              │
  │─paidFetch──▶│                │                    │                   │              │
  │              │─GET───────────▶│                   │                   │              │
  │              │                │─request──────────▶│                   │              │ (seller)
  │              │                │   402+reqts       │                   │              │
  │              │                │◀──────────────────│                   │              │
  │              │                │─onBeforePayment──▶│                   │              │
  │              │                │                   │─routes/balance───▶│              │
  │              │                │                   │─quote────────────▶│              │
  │              │                │                   │─approve+swap────────────────────▶│
  │              │                │                   │─pollFill─────────▶│              │
  │              │                │◀──────────────────│ done              │              │
  │              │                │─paymentPayload───▶│                   │              │
  │              │                │─retry w/ X-PAYMENT▶                   │              │ (seller)
  │              │◀───────────────│ 200 + result      │                   │              │
  │◀─response   │                │                    │                   │              │
```

## Step by step

### 1. Hook registration

When you call `createX402WithAcross`, a callback is registered on the underlying x402 client via `x402Client.onBeforePaymentCreation`. The callback runs *after* the seller has returned a 402 with payment requirements, but *before* the x402 SDK constructs the actual payment payload.

### 2. Hook entry

The callback receives `{ selectedRequirements }`, which contains at least:

- `network` — e.g. `"eip155:8453"`
- `asset` — destination ERC-20 contract address
- `amount` — required amount in the asset's smallest unit, as a string

If `selectedRequirements.network` does **not** start with `eip155:`, the hook returns immediately without bridging. (This SDK is EVM-only.)

### 3. Destination balance check

The hook calls `getErc20Balance(buyer, asset, destinationRpc)` to read the buyer's balance for `asset` on the destination chain.

- If `forceBridge === false` and `destinationBalance >= destinationAmount`, the hook returns and **no bridging happens**. `bridgeInfo.used` is `false`.

### 4. Compute shortfall

```
if forceBridge:
  shortfall = destinationAmount + gasBuffer
else:
  shortfall = destinationAmount - destinationBalance + gasBuffer
```

The `gasBuffer` (default `1000n`, in input-token smallest units) provides a small cushion so the buyer has headroom on the destination after the bridge fills.

### 5. Route discovery

The hook calls `GET https://app.across.to/api/available-routes?destinationChainId=<id>` and filters results:

- Same `destinationToken` as the required `asset` (case-insensitive compare).
- `originChainId !== destinationChainId`.
- `originChainId` included in `originChainIds` (default `[42161, 10, 137, 1, 8453]`).

### 6. Origin selection — fast path

If `preferredOriginChainId` is set:

- Find a route from that origin in the filtered candidate list.
- Read the buyer's balance of that route's origin token on that chain.
- **If balance > 0** → use this route (skip parallel scan).
- **If balance is 0** AND `preferredOriginChainOnly === true` → throw `Preferred origin chain X has no funded route`.
- **If balance is 0** AND `preferredOriginChainOnly === false` → fall through to dynamic scan.
- **If no route exists from preferred chain** AND `preferredOriginChainOnly === true` → throw `No Across route from preferred chain X to chain Y`.
- **If no route exists from preferred chain** AND `preferredOriginChainOnly === false` → fall through to dynamic scan.

### 7. Origin selection — dynamic scan

If no fast-path winner:

- Read buyer balances on **all** candidate origin chains in parallel (`Promise.allSettled`).
- Drop any chain where balance is 0 or the read failed.

Then:

- **Exactly one funded chain** → use it directly.
- **More than one funded chain**:
  1. Request Across quotes in parallel for each (using `tradeType=exactOutput`).
  2. For each, verify `quote.inputAmount <= balance`. Discard if not (the chain doesn't have enough to cover fees).
  3. Sort surviving candidates by:
     1. Balance **descending** (prefer the chain with the most slack).
     2. Fee in USD **ascending** (prefer cheaper).
     3. Origin chain id **ascending** (deterministic tiebreaker).
  4. Take the first.

If **no candidate** survives → throw `No Across origin route with sufficient funds found for <network> <token>`.

### 8. Bridge execution

For the chosen route:

1. If the quote wasn't already obtained, fetch a fresh one from `GET /api/swap/approval`.
2. Verify `originBalance >= quote.inputAmount`, else throw `Insufficient origin balance. Need X on chain Y, have Z`.
3. Create a viem `walletClient` and `publicClient` on the origin RPC.
4. Send each tx in `quote.approvalTxns` (typically ERC-20 `approve`), waiting for each receipt.
5. Send `quote.swapTx` (the Across deposit). Wait for receipt.
6. Poll `GET /api/deposit/status` every **2,000 ms**, up to **120,000 ms**, until `status === 'filled'`.
   - On `expired` → throw `Across deposit expired without fill`.
   - On timeout → throw `Across fill timed out after 120s`.

### 9. Populate `bridgeInfo`

After a successful bridge, the hook writes:

```
bridgeInfo.used = true
bridgeInfo.forced = forceBridge
bridgeInfo.originChainId = <chosen>
bridgeInfo.destinationChainId = <destination>
bridgeInfo.inputToken = <origin token>
bridgeInfo.outputToken = <destination token>
bridgeInfo.inputAmount = quote.inputAmount
bridgeInfo.destinationAmount = <required amount>
bridgeInfo.depositTxHash = <origin tx>
bridgeInfo.fillTxHash = <destination tx>
```

If the hook returned early because the destination was already funded, `used` stays `false` and only `forced`, `destinationChainId`, `outputToken`, `destinationAmount` are populated.

### 10. x402 builds payment

Control returns to x402, which now sees a funded destination, constructs the EIP-712 payment payload, and replays the request with the `X-PAYMENT` header. The seller verifies and (typically) settles on-chain.

## Notes on concurrency

`bridgeInfo` is a **single object mutated per request**. If you make concurrent paid requests with the same `paidFetch`, they will overwrite each other's `bridgeInfo`. Either:

- Await sequentially (and snapshot `bridgeInfo` between calls with `{ ...bridgeInfo }`), or
- Construct one `createX402WithAcross` client per concurrent caller.

## What the hook does NOT do

- It does **not** modify the x402 payment payload — the x402 SDK still builds and signs that.
- It does **not** verify or settle the payment — that's the seller's responsibility.
- It does **not** sweep destination funds back to the origin; bridged funds stay on the destination chain.
- It does **not** retry failed Across deposits automatically — your application must retry the entire `paidFetch` call.
