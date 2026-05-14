# Tuning Bridge Behavior

How to optimize for speed, cost, predictability, and testing.

## Goal: minimize latency

The slowest part of any paid call is bridge fill (~10–60 s). Eliminating bridging when not needed is the biggest win; speeding up the bridge selection when it IS needed is the second.

### Eliminate the bridge entirely

If the buyer already has destination funds, the hook returns immediately and no bridge happens.

- Keep the buyer pre-funded on the destination chain.
- Only let it dip below the per-call cost when you want a top-up.

### Pin the origin chain (fast path)

When bridging IS needed, route discovery + balance scan + multi-chain quoting takes time. If you know the buyer is funded on a particular chain:

```js
across: {
  preferredOriginChainId: 42161,   // Arbitrum
}
```

The fast path reads one balance, skips the parallel scan, and avoids comparing quotes across multiple origins.

### Pin strictly (no fallback)

If you want to fail fast rather than scan when the preferred chain is empty:

```js
across: {
  preferredOriginChainId: 42161,
  preferredOriginChainOnly: true,
}
```

This throws if Arbitrum is unfunded or has no route, instead of scanning others.

## Goal: minimize fees

Across charges a fee on each bridge. The SDK picks routes by:

1. Highest balance (favors the chain with the most slack).
2. Then lowest USD fee.
3. Then deterministic ordering by chain id.

If you want to favor *only* low fees, narrow `originChainIds` to chains you know are cheap (drop Ethereum), or pin via `preferredOriginChainId`.

## Goal: avoid landing on the destination with no gas headroom

When `gasBuffer` is too low, the buyer can arrive on the destination with exactly the required amount and no headroom for the next call's gas. To raise the headroom:

```js
across: {
  gasBuffer: 50_000n,  // tune to your asset's decimals
}
```

`gasBuffer` is denominated in the **input** token's smallest unit:

- USDC has 6 decimals, so `1_000n = 0.001 USDC`, `50_000n = 0.05 USDC`.
- USDT also has 6 decimals.
- ETH (if you were bridging WETH) has 18 decimals.

The default `1000n` is tiny — fine for USDC dust but worth raising for production.

## Goal: deterministic testing

In CI or dev environments, you usually want the bridge path to actually exercise even when the destination is funded:

```js
across: {
  forceBridge: true,
}
```

Every paid request will deposit and fill regardless of destination balance. `bridgeInfo.forced` is set to `true` so your tests can assert on it.

## Goal: production-grade RPCs

Defaults are public free RPCs that will rate-limit you in production. Override:

```js
across: {
  rpcs: {
    8453:  process.env.BASE_RPC,    // Alchemy, QuickNode, dRPC, etc.
    42161: process.env.ARB_RPC,
    10:    process.env.OP_RPC,
    137:   process.env.POLYGON_RPC,
    1:     process.env.ETH_RPC,
  },
}
```

Unspecified chains continue to use defaults. At minimum override the destination chain (most reads happen there) and any preferred origin.

## Common config combinations

```js
// Production, single-origin buyer
{
  preferredOriginChainId: 42161,
  rpcs: { 8453: '...', 42161: '...' },
  gasBuffer: 50_000n,
}

// Production, multi-origin buyer
{
  rpcs: { 8453: '...', 42161: '...', 10: '...', 137: '...', 1: '...' },
  gasBuffer: 50_000n,
}

// Test the crosschain path even when destination has funds
{
  forceBridge: true,
  verbose: true,
}

// Strict — only Arbitrum, fail otherwise
{
  preferredOriginChainId: 42161,
  preferredOriginChainOnly: true,
  rpcs: { 42161: '...', 8453: '...' },
}

// Local dev with extra logs
{
  verbose: true,
}
```

## What to monitor in production

- **Average bridge fill time** — if `bridgeInfo.fillTxHash` is consistently slow, look at relayer health on https://docs.across.to/ or pin to a different origin.
- **Quote rejection rate** — if `Across quote failed` errors are frequent, the chosen origin may not have liquidity for that route at that amount.
- **`bridgeInfo.used` ratio** — high ratio means you're bridging on most calls; consider keeping more pre-funded on the destination.
