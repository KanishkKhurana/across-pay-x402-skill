# Across Protocol API

The SDK uses three endpoints from `https://app.across.to/api`. You do not need to call them directly — they are documented here so you can understand exactly what's happening, debug responses, or hit them manually when investigating.

## Base URL

```
https://app.across.to/api
```

---

## `GET /api/available-routes`

Discover which origin chains can bridge a given asset to a given destination.

### Query parameters set by the SDK

| Param | Required | Description |
|---|---|---|
| `destinationChainId` | yes | EVM chain id of the destination |

### Filtering applied by the SDK

After fetching the raw response, the SDK filters routes:

1. `destinationToken.toLowerCase() === outputToken.toLowerCase()` (match the x402 requirement's `asset`).
2. `originChainId !== destinationChainId` (no same-chain "bridge").
3. `originChainId` is in `originChainIds` (default `[42161, 10, 137, 1, 8453]`).

### Response item shape

```ts
type AcrossRoute = {
  originChainId: number;
  originToken: string;
  originTokenSymbol: string;
  destinationChainId: number;
  destinationToken: string;
  destinationTokenSymbol: string;
};
```

The SDK returns `[]` if the HTTP response is not `ok`.

---

## `GET /api/swap/approval`

Get a quote and the approval+swap transactions for a bridge.

### Query parameters set by the SDK

| Param | Value |
|---|---|
| `originChainId` | chosen origin chain id |
| `destinationChainId` | x402 destination chain id |
| `inputToken` | origin ERC-20 contract address |
| `outputToken` | destination ERC-20 contract address |
| `amount` | shortfall (string, smallest unit) |
| `depositor` | buyer address |
| `recipient` | buyer address (same wallet) |
| `tradeType` | `exactOutput` (so the buyer receives exactly the shortfall) |

### Response shape

```ts
type AcrossQuote = {
  inputAmount: string;           // how much origin token will be spent
  expectedOutputAmount: string;
  minOutputAmount: string;
  expectedFillTime: number;
  approvalTxns: Array<{ chainId: number; to: string; data: string }>;
  swapTx: {
    chainId: number;
    to: string;
    data: string;
    gas: string;
    simulationSuccess: boolean;
  };
  fees: { total: { amount: string; amountUsd: string } };
  id: string;
};
```

### How the SDK uses the quote

- `quote.approvalTxns` — sent one by one via viem `walletClient.sendTransaction`, awaiting receipt each time.
- `quote.swapTx` — sent via viem `walletClient.sendTransaction`, awaiting receipt.
- `quote.fees.total.amountUsd` — parsed as a Number to rank candidates during multi-origin selection (ties broken by chain id).

### Error handling

If the response is not `ok`, the SDK throws:

```
Across quote failed (<status>): <body>
```

---

## `GET /api/deposit/status`

Poll for the destination-chain fill of an Across deposit.

### Query parameters set by the SDK

| Param | Required | Description |
|---|---|---|
| `originChainId` | yes | Chain id where the deposit transaction landed |
| `depositTxHash` | yes | The deposit transaction hash from the swap |

### Response shape

```ts
type DepositStatus = {
  status: 'pending' | 'filled' | 'expired';
  fillTx?: string;
  destinationChainId?: number;
  outputAmount?: string;
};
```

### Polling behavior

| Setting | Value |
|---|---|
| Interval | 2,000 ms |
| Total timeout | 120,000 ms |
| On `filled` | resolve with the full `DepositStatus` |
| On `expired` | throw `Across deposit expired without fill` |
| On timeout | throw `Across fill timed out after 120s` |

Currently the SDK does **not** expose these timing parameters on its public API. If the relayer takes longer than 120 s the call fails — your application must retry the entire `paidFetch`.

---

## Manual debugging

You can hit the same endpoints directly to debug routing problems. Examples:

```bash
# What origins can bridge USDC to Base?
curl "https://app.across.to/api/available-routes?destinationChainId=8453"

# Get a quote for the bridge the SDK would attempt
curl "https://app.across.to/api/swap/approval?originChainId=42161&destinationChainId=8453&inputToken=0xaf88d065e77c8cC2239327C5EDb3A432268e5831&outputToken=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913&amount=10000&depositor=0xYOUR_BUYER&recipient=0xYOUR_BUYER&tradeType=exactOutput"

# Check fill status of a deposit
curl "https://app.across.to/api/deposit/status?originChainId=42161&depositTxHash=0x..."
```
