---
name: openfin-onramp
description: Fiat-to-crypto onramp through OpenFinance — smart-routed across Moonpay (global cards / Apple Pay / Google Pay / bank) and Onramp.money (India via UPI / IMPS). Use whenever the user wants to buy crypto with fiat, top up a wallet, fund a Hyperliquid / Polymarket / Solana wallet, or "I'm out of USDC, how do I get some". Triggers&#58; "buy USDC with my card", "credit card onramp", "Apple Pay USDC", "deposit fiat", "I have no crypto, how do I start", "fund my wallet with fiat", "buy ETH on Base with USD", "deposit ₹1000", "buy USDC with INR", "add ₹5000 USDC", "top up with UPI", "onramp via UPI/IMPS", "India onramp". Default to POST /agent/onramp (smart router — `baseCurrency=inr` → Onramp.money, anything else → Moonpay); use POST /agent/onramp/onrampmoney or POST /agent/onramp/moonpay only to force a provider. Onramp.money's hosted flow is INR-only (UPI/IMPS) and supports a constrained (coin × network) matrix — usdt on bep20|matic20|erc20|trc20, usdc on bep20|matic20 only (NOT erc20 / solana), busd on bep20, matic on matic20, bnb on bep20, eth on erc20|matic20, sol on spl. Calls outside the matrix throw — propose a supported network instead (e.g. "buy USDC with INR on Ethereum" → propose Polygon). Wallet defaults to the caller's OpenFinance-managed wallet for the chain (EVM or Solana). No on-chain signing from the agent — the user opens the returned URL to complete KYC + payment in the provider's UI. Pair with openfin-relay (bridge after onramp lands on a different chain) and openfin-setup (API key check).
---

# Fiat Onramp (Moonpay + Onramp.money)

The user can buy crypto with fiat through OpenFinance. The backend
builds a prefilled hosted-mode URL — the user opens it to complete KYC
and payment. Funds land in the user's OpenFinance-managed wallet for
the chosen chain.

Two providers, one smart-router endpoint:

| Provider | Rails | Best for | `baseCurrency` |
|---|---|---|---|
| **Moonpay** | Cards, Apple Pay, Google Pay, SEPA, bank transfer | Global / non-INR | `usd`, `eur`, `gbp`, … (any non-`inr`) |
| **Onramp.money** | UPI, IMPS, bank transfer | India / INR | `inr` (only) |

The agent never signs an on-chain tx for this. Onramp lives entirely in
the provider's flow.

## Prerequisite

1. User completed `openfin-setup` (API key in place — `open_…`).
2. User's wallet is provisioned for the destination chain (EVM by
   default; Solana wallet must be enabled if `chain=solana` /
   `currencyCode=*_sol`). Onramp.money also supports `tron`.

## Endpoints

### `POST /agent/onramp` — smart router (use this by default)

Picks the provider from `baseCurrency`:

- `baseCurrency: "inr"` → **Onramp.money** (UPI/IMPS)
- anything else (`usd`, `eur`, `gbp`, …, or omitted — defaults `usd`) → **Moonpay**

Body shape is the union of the two providers' fields; the smart router
forwards only the relevant ones. Common fields below; provider-specific
ones in the per-provider sections.

| Field | Type | Notes |
|---|---|---|
| `baseCurrency` | string | **Routing key.** `"inr"` → Onramp.money, else Moonpay. Default `"usd"`. |
| `chain` | string | `ethereum` / `polygon` / `base` / `arbitrum` / `optimism` / `bsc` / `tron` / `solana`. (Onramp.money supports a subset — see matrix.) |
| `currency` | string | Crypto symbol (`usdc`, `usdt`, `eth`, `sol`, `btc`, …). |
| `amount` | number | Fiat amount the user will spend. |
| `walletAddress` | string | Optional — defaults to caller's wallet for the chain. |
| `redirectURL` | string | Provider redirects here after success. |

**Response**

```json
{ "provider": "moonpay" | "onrampmoney", "url": "https://…" }
```

Hand the URL back to the user.

### `POST /agent/onramp/moonpay` — explicit Moonpay

Force the Moonpay provider (e.g. user asks for "Apple Pay" specifically
even when `baseCurrency=inr`).

| Field | Type | Notes |
|---|---|---|
| `chain` | string | One of `ethereum` / `polygon` / `base` / `arbitrum` / `optimism` / `bsc` / `solana`. Combined with `currency` to derive `currencyCode`. |
| `currency` | string | Crypto symbol. Combined with `chain`. |
| `currencyCode` | string | Direct Moonpay code (e.g. `usdc_polygon`). **Overrides** `chain` + `currency` derivation. |
| `amount` | number | Fiat amount the user will spend. |
| `baseCurrency` | string | ISO-4217 fiat code, lowercased. Default `usd`. |
| `walletAddress` | string | Optional — defaults to caller's EVM/Solana address. |
| `redirectURL`, `externalCustomerId`, `theme`, `colorCode` | string | Optional pass-through. |

Email is auto-pulled from the authenticated user record. Response is
`{ provider: "moonpay", url }`.

### `POST /agent/onramp/onrampmoney` — explicit Onramp.money (INR via UPI/IMPS)

Force the Onramp.money provider. INR-only — the hosted flow doesn't
expose other fiats.

| Field | Type | Notes |
|---|---|---|
| `chain` | string | `ethereum` / `polygon` / `bsc` / `tron` / `solana`. Mapped to Onramp's legacy `network` code. |
| `network` | string | Legacy network code (`erc20` / `matic20` / `bep20` / `trc20` / `spl`). Override of `chain` mapping. |
| `currency` / `coinCode` | string | Coin symbol — `usdc`, `usdt`, `busd`, `matic`, `bnb`, `eth`, `sol`. Case-insensitive. |
| `amount` | number | INR the user will spend (e.g. `1000` = ₹1000). |
| `coinAmount` | number | Crypto-side target (alternative to `amount`). When both are passed, Onramp uses `coinAmount`. |
| `paymentMethod` | `1` \| `2` | `1` = UPI / Instant (default in India), `2` = IMPS / Bank. Skip to let the user choose on the form. |
| `walletAddress` | string | Optional — defaults to caller's wallet for the chain. |
| `redirectURL`, `addressTag` | string | Optional pass-through. |

Response is `{ provider: "onrampmoney", url, network, coinCode, walletAddress }`.

## Onramp.money supported (coin × network) matrix

Onramp.money's hosted flow only supports a fixed set of (coin × network)
pairs. **Calls outside this matrix throw** with a clear error naming the
supported networks for the requested coin — propose a supported pair
back to the user instead of retrying.

| Coin | Supported networks |
|---|---|
| `usdt` | `bep20` (BSC), `matic20` (Polygon), `erc20` (Ethereum), `trc20` (Tron) |
| `usdc` | `bep20` (BSC), `matic20` (Polygon) — **NOT** `erc20` / `solana` |
| `busd` | `bep20` |
| `matic` | `matic20` |
| `bnb`  | `bep20` |
| `eth`  | `erc20`, `matic20` |
| `sol`  | `spl` |

Practical examples of how to handle near-misses:

- "Buy USDC with INR on Ethereum" → USDC isn't supported on `erc20` via
  Onramp.money. **Propose Polygon** ("USDC on Polygon via UPI?") or
  switch to Moonpay.
- "Buy USDC with INR on Solana" → not supported. **Propose Polygon or
  BSC**, or switch to Moonpay (`usdc_sol` works there).
- "Buy ETH with INR on Arbitrum / Base / Optimism" → only `erc20` and
  `matic20` are supported on Onramp.money for ETH. Either onramp ETH
  on Ethereum/Polygon and bridge via `openfin-relay`, or use Moonpay.

## Chain → Onramp.money network mapping

The backend maps friendly chain names to Onramp's legacy network codes:

| `chain` | Onramp `network` |
|---|---|
| `ethereum` | `erc20` |
| `polygon` | `matic20` |
| `bsc` | `bep20` |
| `tron` | `trc20` |
| `solana` | `spl` |

Pass `network` directly to override.

## Currency-code rules of thumb (Moonpay)

If the agent passes `chain` + `currency`, the backend derives
`currencyCode`. Common pairs Moonpay supports:

| chain | currency | derived `currencyCode` |
|---|---|---|
| `ethereum` | `eth` | `eth` |
| `ethereum` | `usdc` | `usdc` |
| `polygon` | `usdc` | `usdc_polygon` |
| `base` | `usdc` | `usdc_base` |
| `arbitrum` | `usdc` | `usdc_arbitrum` |
| `optimism` | `usdc` | `usdc_optimism` |
| `solana` | `sol` | `sol` |
| `solana` | `usdc` | `usdc_sol` |

If unsure, pass `currencyCode` directly with the exact Moonpay code.

## Wallet defaults

If `walletAddress` is omitted:

- `chain` is EVM (`ethereum`, `polygon`, `base`, `arbitrum`,
  `optimism`, `bsc`, `tron`) → uses the caller's EVM address
- `chain=solana` or `currencyCode` ends in `_sol` → uses the caller's
  Solana address
- If the required wallet isn't provisioned, the request fails fast —
  surface the error and send the user back to openfinance.tech.

Use `GET /agent/wallets` (see `openfin-setup`) if the agent wants to
display the receiving address before opening the URL.

## Pair with other skills

- After onramp lands on chain X but the user wants to trade on chain
  Y — use `openfin-relay` to bridge.
- After onramp lands as USDC on Arbitrum and user wants to trade on
  Hyperliquid — `GET /agent/trading/deposit-address`, send USDC there
  (no bridge needed).
- After onramp lands as USDC on Polygon and user wants to trade on
  Polymarket — pUSD is the V2 collateral and lives on the Polymarket
  **deposit wallet**, not the EOA. Swap USDC → pUSD and forward to the
  deposit wallet (`GET /agent/polymarket/deposit-wallet`) before
  trading.

## Examples

### USD → USDC on Base via card (Moonpay)

```http
POST /agent/onramp
x-api-key: open_…
Content-Type: application/json

{
  "chain": "base",
  "currency": "usdc",
  "amount": 100,
  "baseCurrency": "usd"
}
```

Response:
```json
{ "provider": "moonpay", "url": "https://buy.moonpay.com/?currencyCode=usdc_base&walletAddress=0x…&baseCurrencyAmount=100&baseCurrencyCode=usd&signature=…" }
```

### INR → USDC on Polygon via UPI (Onramp.money)

```http
POST /agent/onramp
x-api-key: open_…
Content-Type: application/json

{
  "baseCurrency": "inr",
  "chain": "polygon",
  "currency": "usdc",
  "amount": 5000,
  "paymentMethod": 1
}
```

Response:
```json
{ "provider": "onrampmoney", "url": "https://onramp.money/app/?appId=…&coinCode=usdc&network=matic20&walletAddress=0x…&fiatAmount=5000&paymentMethod=1&merchantRecognitionId=…" }
```

### INR → USDT on Tron via IMPS (Onramp.money)

```http
POST /agent/onramp
x-api-key: open_…
Content-Type: application/json

{
  "baseCurrency": "inr",
  "chain": "tron",
  "currency": "usdt",
  "amount": 2000,
  "paymentMethod": 2
}
```

Agent surfaces the URL: "Open this link to complete the IMPS bank
transfer. USDT will land at your Tron wallet once cleared."

## Don't

- **Don't try to use Onramp.money for non-INR fiats.** The hosted flow
  is INR-only — `baseCurrency=usd/eur/gbp/...` must go through Moonpay.
- **Don't ignore the (coin × network) matrix on Onramp.money.** Passing
  an unsupported pair (e.g. `usdc` on `erc20`, `usdc` on `solana`)
  throws — propose a supported network or switch to Moonpay.
- Don't ask the user for credit-card / UPI details. The provider
  collects them in their own KYC-compliant UI; the OpenFinance backend
  never sees them.
- Don't treat the returned URL as authenticated to the user. It's
  prefilled but the actual KYC + payment happen on the provider.
- Don't poll the backend for onramp status. The provider handles the
  buy and emits funds to the wallet directly. To verify funds arrived,
  read on-chain via `openfin-onchain` (`/balances`) or the chain's
  block explorer.

## MCP note

- **`get_onramp_url`** (smart router — primary) — same body as
  `POST /agent/onramp`. Returns `{ provider, url, ... }`. Use this
  by default.
- `get_moonpay_onramp_url` — explicit Moonpay only; same body as
  `POST /agent/onramp/moonpay`.
