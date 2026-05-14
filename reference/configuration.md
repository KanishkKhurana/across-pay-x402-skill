# Configuration Reference

Everything that can be configured, with defaults and behavior notes. All values verified against `across-pay-x402@1.2.0`.

## `createX402WithAcross` config

| Field | Type | Default | Description |
|---|---|---|---|
| `account` | `viem.Account` | (required) | The buyer's viem account. Use `resolveCoinbaseWallet()` or `privateKeyToAccount(...)`. |
| `across` | `AcrossConfig` | `{}` | Crosschain bridging behavior. See table below. |
| `polyfill` | `boolean` | `false` | If `true`, replaces `globalThis.fetch` with the payment-wrapped fetch. |
| `verbose` | `boolean` | `false` | Print `[x402+across]` lines describing each decision. |

## `AcrossConfig`

| Field | Type | Default | Description |
|---|---|---|---|
| `preferredOriginChainId` | `number` | `undefined` | Try this origin chain first ("fast path"). If funded, use it and skip scanning others. |
| `preferredOriginChainOnly` | `boolean` | `false` | Combined with `preferredOriginChainId`: if `true`, throw when the preferred chain is unfunded or unrouted instead of scanning fallbacks. |
| `originChainIds` | `number[]` | `[42161, 10, 137, 1, 8453]` | Whitelist of origin chains to consider during dynamic scan. |
| `gasBuffer` | `bigint` | `1000n` | Extra units added to the shortfall when bridging, so the buyer also has a small cushion on the destination after the fill. Denominated in the **input** token's smallest unit. |
| `forceBridge` | `boolean` | `false` | Always bridge, even if the destination chain already has enough. Useful for testing the crosschain path end-to-end. |
| `rpcs` | `Record<number, string>` | (see defaults) | Override RPC URLs per chain id. Merged on top of built-in `DEFAULT_RPCS`. |

## Default RPCs

Built into the SDK:

| Chain ID | Name | RPC URL |
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

For any chain id **not** in this map, the SDK falls back to:

```
https://lb.drpc.org/ogrpc?network=<chainId>
```

This fallback is public and may be rate-limited. For production usage, override every chain you depend on by passing `across.rpcs`.

## Environment variables

The `across-pay-x402` package itself **does not read any environment variables directly**. Env vars matter in exactly two contexts:

### CDP wallet (`resolveCoinbaseWallet()`)

Read by `@coinbase/cdp-sdk`:

| Var | Required | Purpose |
|---|---|---|
| `CDP_API_KEY_ID` | yes | CDP API key identifier |
| `CDP_API_KEY_SECRET` | yes | CDP API key secret |
| `CDP_WALLET_SECRET` | yes | Required to sign with the server wallet |

### Local PK wallet

If you use `privateKeyToAccount`, the conventional pattern:

| Var | Required | Purpose |
|---|---|---|
| `BUYER_PK` | yes (when using local key) | `0x`-prefixed private key. Never commit. Use `.env.local` or a secret manager. |

This name is a convention only — the SDK never reads it directly.

## When defaults are right vs. when to override

| Situation | Recommended config |
|---|---|
| Buyer is reliably funded on Arbitrum | `preferredOriginChainId: 42161` |
| Buyer is strictly only funded on one chain | `preferredOriginChainId: <id>, preferredOriginChainOnly: true` |
| Buyer is funded on a chain not in defaults | `originChainIds: [...defaults, <newChain>]` and probably `rpcs: { <newChain>: '<rpcUrl>' }` |
| Testing crosschain path with funds already on destination | `forceBridge: true` |
| Buyer often lands short on destination gas | Raise `gasBuffer` (e.g. `50_000n` for USDC) |
| Default public RPCs are rate-limiting | Provide private RPCs via `rpcs` |
| Production usage | Always: provide your own `rpcs` for at least the destination chain and any preferred origin |
