# Wallet Options

Two ways to provide the buyer signer. Pick one based on key custody preference.

## Option A — Coinbase CDP server wallet (recommended for production)

Keys are managed inside Coinbase Developer Platform infrastructure. Your code never sees the private key.

### Setup

1. Create a CDP account at https://www.coinbase.com/developer-platform.
2. Create an API key and capture `CDP_API_KEY_ID` and `CDP_API_KEY_SECRET`.
3. Generate a wallet secret and store it as `CDP_WALLET_SECRET`.

### Usage

```js
import { resolveCoinbaseWallet } from 'across-pay-x402';

const { account, provider, source } = await resolveCoinbaseWallet();
// account is a viem Account — pass it to createX402WithAcross.
```

### Required env vars

| Var | Purpose |
|---|---|
| `CDP_API_KEY_ID` | CDP API key identifier |
| `CDP_API_KEY_SECRET` | CDP API key secret |
| `CDP_WALLET_SECRET` | Required for signing transactions |

### Notes

- `resolveCoinbaseWallet()` calls `cdp.evm.getOrCreateAccount({ name: 'x402-buyer' })`. The wallet is identified by the fixed name `"x402-buyer"`; subsequent calls return the same wallet. The address is stable — fund once and reuse.
- Signing is API-side: every signature round-trips to CDP. Expect a small latency overhead per signing.
- The CDP wallet is per-account, not per-project. If you use the same CDP credentials from a different project, you'll get the same `x402-buyer` wallet.

## Option B — Local private-key wallet

You hold and manage the private key yourself. Useful for local dev, automated tests, or when you don't want a CDP dependency.

### Usage

```js
import { createX402WithAcross } from 'across-pay-x402';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount(process.env.BUYER_PK); // 0x-prefixed
```

### Security

- **Never** hard-code the private key.
- **Never** commit it. Use `.env.local`, a secret manager, or process env.
- For production, prefer a managed signer (CDP, AWS KMS, Fireblocks, Turnkey, etc.).
- The PK gives full control of the wallet's funds — treat it like cash.

## Comparison

| Property | CDP server wallet | Local PK |
|---|---|---|
| Key custody | Coinbase | You |
| Setup steps | CDP account + 3 env vars | Generate or import a key |
| Best for | Production, autonomous agents | Local dev, tests |
| Latency | One extra API call per signing | In-process |
| Audit trail | CDP logs | None |
| Revocable | Rotate CDP API key | Move funds to a new wallet |

## Verifying a wallet is configured correctly

CDP wallet:

```js
const { account } = await resolveCoinbaseWallet();
console.log('Address:', account.address);
```

Local PK wallet:

```js
import { privateKeyToAccount } from 'viem/accounts';
const account = privateKeyToAccount(process.env.BUYER_PK);
console.log('Address:', account.address);
```

Then send a few cents of USDC to that address on **Arbitrum** (or any default origin chain) before running anything that calls `paidFetch`.

## Switching wallets

The wallet is just a viem `Account` — `createX402WithAcross` doesn't care where it came from. You can swap between CDP and PK signers without changing any other code.

```js
const account = process.env.USE_LOCAL_KEY
  ? privateKeyToAccount(process.env.BUYER_PK)
  : (await resolveCoinbaseWallet()).account;

const { fetch: paidFetch } = createX402WithAcross({ account });
```
