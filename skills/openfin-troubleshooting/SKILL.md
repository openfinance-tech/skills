---
name: openfin-troubleshooting
description: >-
  Error-to-fix playbook for every known failure mode on the OpenFinance backend
  — Polymarket, Relay, Hyperliquid, Solana RPC, and account-setup issues.
  Use this the moment a call fails, returns an unexpected status, or behaves
  inconsistently with on-chain state. Triggers on ANY of these error signatures
  verbatim or in paraphrase. Polymarket&#58; "maker address not allowed,
  please use the deposit wallet flow", "not enough balance / allowance:
  balance: 0" after the deposit-wallet migration (pUSD on EOA instead of
  deposit wallet), "401 invalid authorization" on POST /agent/polymarket/approvals
  (relayer rejects unauthed RelayClient), "allowance: 0 but on-chain shows
  max", "CLOB reports allowance 0", "approvals confirmed but order rejected",
  "404 upstream" on market orders, "tick size" rejection, "order size below
  minimum", USDC.e vs pUSD vs native USDC confusion, V1 vs V2 exchange
  confusion, EOA-vs-deposit-wallet maker confusion. Relay&#58; "InstructionFallbackNotFound", "Custom:101",
  "Custom:6000", "AnchorError", "Blockhash not found", "TransactionExpired",
  "No valid authorization signatures were provided", 412 setup-incomplete
  errors on Solana origin, quote succeeded but execute failed, stuck funds
  on Solana, stuck funds cross-chain, topupGas forced off.
  Hyperliquid&#58; "Insufficient margin", "account value too low", "price out
  of bounds", "withdrawal not arriving on Arbitrum", WebSocket stale data,
  spot vs perp balance confusion (unified account — spot USDC fungibly
  backs perp orders, agent must sum both).
  General&#58; any "why is X failing", "why does on-chain and API state
  disagree", "what does this error mean". Read this BEFORE assuming a bug
  in the MCP or backend — most of these errors are already catalogued with
  known fixes.
---

# OpenFinance Troubleshooting

Error → diagnosis → fix. Check here before assuming an issue is a server-side
bug: most failures are known and have been hit before.

## Polymarket

### `maker address not allowed, please use the deposit wallet flow`

**Root cause:** Polymarket's CLOB rejects raw EOAs as makers. New accounts
must trade via a per-EOA "deposit wallet" — a deterministic CREATE2 smart
contract owned by the EOA, pre-deployed by Polymarket's relayer. The order
must name the deposit wallet as `funder` (and `signer`) with
`signatureType=POLY_1271` (EIP-1271 verified on-chain), not the EOA.
Pre-existing wallets that traded before the migration may have been
allowlisted under the old EOA path; only new wallets hit this.

**Fix:** Already wired in the backend — `polymarketOrder.service.ts`
derives the deposit wallet via `@polymarket/builder-relayer-client`'s
`deriveDepositWallet` and passes `signatureType: SignatureTypeV2.POLY_1271,
funderAddress: <depositWallet>` to the `ClobClient`. Requires
`@polymarket/clob-client-v2 >= 1.0.3` (which adds POLY_1271 + DepositWallet
EIP-712 domain). If still seeing this error, the deployment is behind.
Confirm the user's deposit-wallet address via
`GET /agent/polymarket/deposit-wallet`.

### `not enough balance / allowance: balance: 0` after the deposit-wallet migration

**Root cause:** pUSD is on the EOA, not the deposit wallet. Under the
deposit-wallet flow, orders settle from the deposit wallet — collateral
sent directly to the EOA address is stranded for trading. Common after
onramp/bridge flows that still default to the EOA, or for wallets funded
before the migration.

**Fix:** Resolve the deposit wallet via `GET /agent/polymarket/deposit-wallet`
(returns `{eoa, depositWallet, deployed, pUSD: {eoa, depositWallet}}`).
If `pUSD.eoa > 0` and `pUSD.depositWallet == 0`, transfer pUSD from EOA to
the deposit wallet (regular ERC-20 `transfer`, EOA pays MATIC ~$0.01).
Going forward, route all Polymarket-bound deposits (onramp destinations,
Relay quotes targeting Polygon for trading) to the deposit wallet, never
the EOA. Surface the deposit wallet as "your Polymarket address" — the
EOA is just the signer.

### `POST /agent/polymarket/approvals` returns `401 invalid authorization` from upstream relayer

**Root cause:** Approvals are now set on the deposit wallet via Polymarket's
relayer (`relayer-v2.polymarket.com`) as a meta-tx, not a normal EOA call.
The relayer requires builder-API auth headers
(`POLY_BUILDER_API_KEY/TIMESTAMP/PASSPHRASE/SIGNATURE`). If the
`RelayClient` was instantiated without a `BuilderConfig`, requests go
unauthed and the relayer returns 401. Read calls (e.g. CLOB `/orders`)
work because they use the user's L2 CLOB creds, not the relayer.

This also matches a related class of issue where the CLOB cred ↔ deposit
wallet binding doesn't always provision cleanly for newly deployed deposit
wallets ("signer has to be API key" / `/onboard` succeeds but `/order`
rejects).

**Fix (two paths):**

1. **Add builder auth to RelayClient** — wrap `localBuilderCreds`
   (`{key, secret, passphrase}`) in `BuilderConfig` and pass it to
   `new RelayClient(url, chainId, wallet, builderConfig)`. Mint creds via
   `clob.createBuilderApiKey()` (signed L1 by an EOA authorized for
   `POLYMARKET_BUILDER_CODE`); cache per-user in MongoDB or use a single
   shared admin set. This keeps approvals gas-free.
2. **Skip the relayer** — submit the deposit wallet's batch tx directly
   from the EOA (EOA pays gas ~$0.05). No relayer auth needed; trades
   off gas-free for self-contained signing.

If `pUSD.depositWallet > 0` but allowances are 0, the user is one approvals
batch away from being able to trade — don't place orders before approvals
land or they'll fail at the CLOB or settle-fail on-chain.

### `POST /agent/polymarket/approvals` shows max but CLOB returns `allowance: 0`

**Root cause:** approval targeted the legacy token or legacy exchange spender.
Polymarket migrated the CLOB to V2 — settlement is now against **pUSD**
(`0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB`) on the **V2 Exchange**, not
USDC.e (`0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`) on the V1 Exchange
(`0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E`). Verify with
`curl https://clob.polymarket.com/version` → `{"version": 2}`. Old USDC.e/V1
allowances do **not** carry over; approvals predating the migration look
"max" on the V1 spender but the V2 CLOB checks the pUSD allowance against
the V2 spender, which is `0`.

**Fix:** Re-call `POST /agent/polymarket/approvals` (with `{negRisk: true}`
for neg-risk markets). The server approves pUSD against the V2 Exchange +
NegRiskAdapter / NegRiskExchange + ConditionalTokens spenders. If the
wallet still holds USDC.e (no longer the settlement token), convert /
bridge it to pUSD before trading — the order endpoint will reject otherwise.
This was working with USDC.e/V1 prior to the migration; if a flow that
worked in April 2026 stopped working afterward, this is the cause.

### `POST /agent/polymarket/order/market` returns `404 upstream` (limit orders work)

**Root cause:** SDK's `createMarketOrder` calls `ensureBuilderFeeRateCached`
which does unauthenticated `GET /fees/builder-fees/<code>`. That endpoint
404s until Polymarket's fee service has the builder code onboarded.

**Fix:** Already patched — the backend catches the 404 and caches a zero-rate
entry. If you still see this, the deployment is behind. Retry passes through
after the patch is live.

### `Order failed: tick size` on a price that looks right

**Root cause:** Market's minimum tick size is smaller than `0.01` (the default).
Passing `0.01` when the market needs `0.001` or `0.0001` rejects.

**Fix:** Pass `tickSize: "0.001"` or `"0.0001"` explicitly in the request body.
Read the market's min tick from `GET /agent/polymarket/markets/cid/:conditionId`.

### `Order size below minimum`

Polymarket requires ~$1 USDC minimum notional. `price=0.05, size=10` = $0.50
notional → rejected.

**Fix:** Increase size so `price * size >= 1`.

## Relay

### `InstructionFallbackNotFound (Custom:101)` on Solana-origin execute

**Root cause:** Instruction data decoded with wrong encoding. Relay returns
Solana instruction `data` as **hex** (no `0x` prefix), not base64. Early
backend versions decoded as base64 → garbage bytes → depository rejected.

**Fix:** Already patched. First 8 bytes of a correctly-decoded instruction
should match Relay's discriminator (e.g. `0b9c60da27a3b413`).

### `Blockhash not found` repeatedly on Solana execute

**Root cause:** Cross-RPC divergence — the blockhash was issued by one
RPC and the broadcast went via another that hasn't seen it yet.

**Fix:** Already patched — backend signs and broadcasts via its own
RPC with `skipPreflight: true`. If you still see it, set
`SOLANA_RPC_URL` to a dedicated provider (Helius / Triton / QuickNode)
— the public `api.mainnet-beta.solana.com` is heavily throttled.

### `401 No valid authorization signatures` / `412` on Solana execute

**Root cause:** The user's account at openfinance.tech is missing the
Solana side of the setup. The backend handles signing internally, but
without that setup the Solana signing path can't run.

**Fix:** Tell the user to complete (or re-run) account setup at
[openfinance.tech](https://openfinance.tech). After that, retry
`/agent/relay/execute` — quotes were unaffected, so no state was
spent.

### `POST /agent/relay/execute` says "User has no Solana wallet provisioned"

**Root cause:** The user's account doesn't have a Solana wallet yet
(setup completed before Solana support was added, or Solana isn't
enabled for their account).

**Fix:** Send the user to [openfinance.tech](https://openfinance.tech)
to complete Solana setup, then retry.

### Gas topup forced off unexpectedly

**Expected behavior, not a bug.** When either origin or destination chain ID
is Solana (`792703809`), `topupGas` is forced to `false` server-side. Topup
doesn't work cleanly for EVM↔SVM routes.

## Hyperliquid

### `Insufficient margin` / `account value too low`

**Root cause:** The wallet's USDC is below the required notional, OR
the wallet is not yet in `unifiedAccount` mode (so spot USDC isn't
counted as margin), OR the agent only checked `/account.withdrawable`
and missed the spot-side slice.

**Fix:** Run the auto-unify rule from `openfin-hyperliquid`, then
compute the unified balance:

```
account = GET /agent/trading/account
spot    = GET /agent/trading/account/spot
spotUSDC = spot.balances.find(b => b.coin === 'USDC')?.total ?? 0

if account.withdrawable > 0:
  mode = GET /agent/trading/abstraction
  if mode in ("default", "disabled"):
    POST /agent/trading/abstraction { abstraction: "u" }   # one-shot, idempotent

unifiedFreeUSDC = account.withdrawable + spotUSDC
```

Only call abstraction when `withdrawable > 0` (it's the case where
perp-side USDC may still be split-pool and needs the upgrade). When
`withdrawable === 0` and `spotUSDC > 0`, the wallet is already
unified — that USDC sitting on spot IS withdrawable balance. When
both are zero, there's nothing to unify; tell the user to deposit.

If `unifiedFreeUSDC >= required`, just **place the order**. There's no
transfer step; the unified balance fungibly backs perp, spot, and
withdrawals.

If `unifiedFreeUSDC < required`, walk the funding ladder in
`openfin-hyperliquid`:

1. Bridge from another chain via `openfin-relay` (read `/agent/wallets`
   first to find USDC on Base / Arbitrum / Polygon / Solana).
2. Onramp only if every wallet is empty.

**Counter-examples seen in the wild:**

- Agent reads `account.withdrawable = $0`, ignores `/account/spot`
  (`$18.98 USDC` sitting there), and tells the user to onramp. Wrong
  — sum both endpoints first and place the order.
- Agent sees a wallet on `"default"` abstraction with USDC on both
  sides, treats them as one balance, and places an order that fails
  with `Insufficient margin`. Wrong — until the wallet is upgraded to
  `unifiedAccount`, spot USDC is **not** perp margin. Run
  `set_user_abstraction({abstraction: "u"})` first.

### `Withdrawal not arriving on Arbitrum`

**Expected, not a bug.** `POST /agent/trading/withdraw` → `withdraw3`
finalizes on Arbitrum in **~5 minutes** once L1 validators co-sign. A
flat **$1** is deducted as the bridge fee. If still not visible after
~10 minutes, check Hyperliquid's withdrawal status via the wallet's
explorer page on Hyperliquid; on confirmed delays, the support route is
Hyperliquid's Discord, not the OpenFinance backend.

### WebSocket data is stale

**Root cause:** The backend maintains one shared connection to
`wss://api.hyperliquid.xyz/ws` and caches per-channel. If the connection dropped
and reconnected, the first few messages after reconnect may be missing.

**Fix:** The backend auto-reconnects after 3s. Retry the read; cache fills
within a few seconds. If persistent, something is wrong with outbound WS from
the server host.

### `Order rejected: price out of bounds`

**Root cause:** Hyperliquid rejects prices too far from the current mid (price
bands). Typically ±5-10% depending on asset.

**Fix:** Read mid via `GET /agent/trading/market/mids` and set price within
bounds. For aggressive fills, use limit orders crossing the book but within
the band.

## General principles

1. **Check `GET /agent/wallets` first** when anything auth-related fails. It
   tells you which chains are provisioned for the user; if a chain is
   missing, send them to openfinance.tech to complete setup.
2. **On-chain state != CLOB/Relay state.** Approvals can show on-chain but
   still report 0 upstream because of token/contract mismatches. Don't trust
   "on-chain says X" without also checking the service's view.
3. **When in doubt, read the error string verbatim** — don't paraphrase. Many
   errors have specific signatures (`Custom:101` vs `Custom:6000`) that map to
   exactly one fix.
