# Supported Chains

Chains with built-in RPCs and chains in the default origin set.

## `DEFAULT_RPCS`

These chains have built-in RPC URLs and can be used as origin OR destination with no extra configuration:

| Chain ID | Name | Built-in RPC URL |
|---|---|---|
| 1 | Ethereum Mainnet | `https://eth.llamarpc.com` |
| 10 | Optimism | `https://mainnet.optimism.io` |
| 56 | BNB Smart Chain | `https://bsc-dataseed.binance.org` |
| 137 | Polygon | `https://polygon-rpc.com` |
| 324 | zkSync Era | `https://mainnet.era.zksync.io` |
| 8453 | Base | `https://mainnet.base.org` |
| 42161 | Arbitrum One | `https://arb1.arbitrum.io/rpc` |
| 59144 | Linea | `https://rpc.linea.build` |
| 534352 | Scroll | `https://rpc.scroll.io` |

These are public, free, and rate-limited. For production usage, **override them** with paid endpoints (Alchemy, Infura, dRPC, QuickNode, etc.) via `across.rpcs`.

## `DEFAULT_ORIGIN_CHAINS`

When auto-discovering an origin for the bridge, the SDK only considers these chains:

```js
[42161, 10, 137, 1, 8453]
```

Mapped to names:

| Position | Chain ID | Name |
|---|---|---|
| 1 | 42161 | Arbitrum One |
| 2 | 10 | Optimism |
| 3 | 137 | Polygon |
| 4 | 1 | Ethereum Mainnet |
| 5 | 8453 | Base |

Order does not affect selection — selection is by funded balance and quote fee, not list order.

To add more origin chains, pass `across.originChainIds`:

```js
across: {
  originChainIds: [42161, 10, 137, 1, 8453, 324, 59144, 534352],
}
```

## Destination chains

Any EVM chain that Across supports as a destination can be a destination here, as long as the x402 seller advertises it in the `network` field of its 402 response (using the `eip155:<chainId>` format).

The package's stated mainnet target is **Base (`eip155:8453`)**.

## Fallback RPC

If you point the SDK at a chain not in `DEFAULT_RPCS` and you don't supply a custom RPC, it falls back to:

```
https://lb.drpc.org/ogrpc?network=<chainId>
```

This is a public load-balanced RPC and may be rate-limited. Prefer to supply your own.

## Route availability

Even if a chain is in the SDK's default list, a specific bridge route (e.g. USDC on origin → USDC on Base) only exists if Across actually supports it. Always check `/api/available-routes` (or run with `verbose: true` and look at the `[route discovery]` log line) before assuming a route exists for a given asset.

## Recommended production RPC overrides

At minimum, override the destination chain and any preferred origin:

```js
across: {
  rpcs: {
    8453:   process.env.BASE_RPC,
    42161:  process.env.ARB_RPC,
    10:     process.env.OP_RPC,
    137:    process.env.POLYGON_RPC,
    1:      process.env.ETH_RPC,
  },
}
```
