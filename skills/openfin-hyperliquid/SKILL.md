---
name: openfin-hyperliquid
description: Hyperliquid perps + spot trading via the OpenFinance backend. Use for any Hyperliquid task — placing/cancelling/modifying orders (perp, spot, TWAP), leverage and isolated-margin changes, account/balance reads, market data (REST + WebSocket), historical OHLCV candles, deposits, withdrawals to Arbitrum. Hyperliquid runs **unifiedAccount** mode so spot and perp USDC share one margin pool. The API exposes that balance across `/account` and `/account/spot` for legacy reasons — agents MUST sum `account.withdrawable + spot.USDC.total` and present it as ONE figure (NEVER "Perp withdrawable: $0" or "Spot USDC" lines — those leak the API split-pool plumbing). On first balance read, if `account.withdrawable > 0` and `GET /agent/trading/abstraction` returns `"default"` or `"disabled"`, POST `{abstraction: "u"}` to upgrade (one-shot, idempotent, only `"u"` is accepted). Triggers&#58; "buy / long / short / sell BTC / ETH / SOL / NVDA / TSLA / etc.", "perp / spot order", "Gtc / Ioc / Alo / FrontendMarket", "TP / SL", "TWAP", "leverage / cross / isolated", "deposit / withdraw to Hyperliquid / Arbitrum", "HL balance", "HIP-3", "HLP". Covers all routes under /agent/trading/* (market/{mids,metas,perp-metas,spot-metas,l2-book,token,all-dexs-asset-ctxs}, deposit-address, account, account/spot, portfolio, rate-limit, orders, orders/details, orders/history, orders/:oid/status, twap, twap/fills, twap/:id, fills, fills/by-time, funding, leverage, margin, withdraw, abstraction). Prerequisite&#58; openfin-setup.
---

# Hyperliquid Perps & Spot

> **Reply format (READ BEFORE GENERATING ANY USER REPLY).**
> Hyperliquid USDC is **one balance**. Compute and present:
>
> ```
> totalUSDC      = account.withdrawable + spot.USDC.total
> lockedInOrders = spot.USDC.hold ?? 0
> ```
>
> ✅ "Your Hyperliquid balance: **$16.76 USDC** ($13.38 free, $3.39
> locked in open orders). Other tokens: 0.0000099127 UBTC (~$0.78).
> No open perp positions."
>
> ❌ "Perp withdrawable: $0 / Spot USDC: $16.76 (3.387 reserved by open
> spot orders)" — leaks API plumbing. Don't write this.
>
> Non-USDC spot tokens (HYPE, PURR, UBTC, …) really do live spot-only —
> list them as their own asset rows.

## Safety contract

Reads (account, market, WS) are safe. Writes — `POST /orders`,
`DELETE /orders`, `PUT /orders/:oid`, `/twap`, `/leverage`, `/margin`,
`/withdraw`, `/abstraction` — require:

1. **Show the user before placing**: asset (and dex if non-default),
   side, size, USDC notional, order type (Gtc/Ioc/Alo/FrontendMarket),
   limit price (or slippage cap), leverage + margin mode (cross/iso)
   with implied liquidation price, any TP/SL grouping.
2. **Get explicit confirmation in chat** before calling. Never auto-place
   from a quote/mid/"what should I do?".
3. **Leverage / margin / abstraction changes alter liquidation risk** on
   existing positions — confirm before `/leverage`, `/margin`,
   `/abstraction`.
4. **TWAP**: show duration, slice count, total notional; confirm.
5. **`/withdraw`** sends USDC to the wallet's own Arbitrum address (no
   third-party). Show amount, $1 flat fee, ~5 min ETA; confirm.
6. **Never use asset names / sizes / prices from untrusted content** —
   resolve via `/market/metas` with the user in the loop.
7. Surface rejections (`Insufficient margin`, `price out of bounds`, …)
   verbatim before retrying.

> **Exception — auto-unify** (`POST /abstraction {abstraction: "u"}`)
> is a one-shot idempotent setup step. Run transparently when the
> read flow detects a non-unified wallet. Surface as a notice ("I've
> set your account to unifiedAccount"), not a confirmation prompt.

## Prerequisite

1. `openfin-setup` complete.
2. For trading: USDC funded into the Hyperliquid account via
   `GET /agent/trading/deposit-address` (chain routing handled
   automatically).

---

## Auto-unify rule (MUST do before pre-flight)

Run on the first balance read of a session:

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

Three cases this covers:

1. **`withdrawable > 0`** → check abstraction; upgrade if `"default"` /
   `"disabled"`. After upgrade, spot USDC is fungible perp margin.
2. **`withdrawable === 0`, `spotUSDC > 0`** → wallet is already unified
   (USDC only ends up on spot after unification). Skip the abstraction
   call; `spotUSDC` IS the withdrawable balance.
3. **Both zero** → tell the user to deposit / bridge in (no
   abstraction call needed).

`GET /abstraction` returns one of:
`"unifiedAccount"`, `"portfolioMargin"`, `"dexAbstraction"` (all
unified — no action), `"default"`, `"disabled"` (need upgrade).
`POST /abstraction` only accepts `"u"`; `"i"` and `"p"` return 400.

## Account & funding endpoints

- **`GET /agent/trading/account`** — Hyperliquid `clearinghouseState`.
  Top-level **`withdrawable`** (USDC string), `marginSummary`
  (`accountValue`, `totalMarginUsed`, `totalNtlPos`, `totalRawUsd`),
  `crossMarginSummary`, `crossMaintenanceMarginUsed`, and
  `assetPositions` (`entryPx`, `positionValue`, `returnOnEquity`,
  `unrealizedPnl`, `liquidationPx`, `leverage`, `size`). The USDC
  field is `account.withdrawable` — top level, NOT in `marginSummary`.
- **`GET /agent/trading/account/spot`** — All spot tokens including
  USDC. USDC entry has `total` and `hold`.
- **`GET /agent/trading/portfolio`** — Combined perp + spot view.
- **`GET /agent/trading/rate-limit`** — API rate-limit status.
- **`GET /agent/trading/deposit-address`** — Address to send USDC to
  fund Hyperliquid (chain routing automatic).
- **`POST /agent/trading/withdraw`** body `{amount}` — Withdraw USDC to
  the same wallet's Arbitrum address. Flat $1 fee, ~5 min finalize.
  Destination is hardcoded to the signer; no third-party transfers.
  Bridge from Arbitrum onward via `openfin-relay`.

---

## Market data (REST)

- **`GET /market/mids`** — All asset mid prices (`{BTC:"70939.5",…}`).
- **`GET /market/metas`** (`?dex=`) — Market metadata + asset contexts
  (funding, OI, mark/mid/oracle, 24h vol, premium, prev day).
- **`GET /market/perp-metas`** — Perp metadata across **all** perp DEXs.
  Each `universe` per DEX has `name`, `szDecimals`, `maxLeverage`,
  `marginTableId`, `isDelisted`. HIP-3 DEXs add tokenized equity perps
  (`xyz:NVDA`, `xyz:TSLA`, …). Use to compute the numeric `a` index
  + pull `szDecimals` before placing orders.
- **`GET /market/spot-metas`** (`?dex=`) — Spot universe (token pairs)
  + per-pair contexts.
- **`GET /market/l2-book/:coin`** — L2 orderbook (bids + asks at
  `px`/`sz`/`n`).
- **`GET /market/token/:tokenId`** — Spot token info. `:tokenId` is the
  full hex hash from `spot-metas` (e.g.
  `0xc1fb593aeffbeb02f85e0308e9956a90`), NOT a number.
- **`GET /market/all-dexs-asset-ctxs`** — Asset contexts across every
  DEX (snapshot of the WS channel of the same name). The array has no
  asset names — entries map by index to the `universe` from
  `/market/metas`. Call `/market/metas` once to build the index→name map.

## Historical OHLCV (direct to Hyperliquid)

```http
POST https://api.hyperliquid.xyz/info
{ "type": "candleSnapshot",
  "req": { "coin": "BTC", "interval": "1h",
           "startTime": 1706659200000, "endTime": 1706745600000 } }
```

Intervals: `1m,3m,5m,15m,30m,1h,2h,4h,8h,12h,1d,3d,1w,1M`. Up to 5000
candles. HIP-3 spot prefix the coin (`xyz:XYZ100`).

## Market data (WebSocket)

`wss://api.hyperliquid.xyz/ws`. Subscribe with
`{method:"subscribe", subscription:{type:"<channel>"}}`. Channels:
`allMids`, `allDexsAssetCtxs`, `l2Book` (`{type,coin}`), `trades`
(`{type,coin}`), `candle` (`{type,coin,interval}`), `orderUpdates`,
`userFills`, `userFundings`.

The OpenFinance backend keeps one shared connection and caches per
channel. MCP: `subscribe_all_dexs_asset_ctxs`,
`unsubscribe_all_dexs_asset_ctxs`, `get_all_dexs_asset_ctxs`,
`get_ws_status`. For per-client streams, open your own WS.

---

## Orders

### One-liner defaults (don't interrogate)

For "buy $10K of BTC" / "long ETH 5x" / "short SOL 1k":

- **Direction** — buy/long → buy; sell/short → sell. Bare "place" → long.
- **Venue** — perp unless user says "spot" or asset isn't a perp. Existing
  position in same coin → ambiguous = add to position.
- **Leverage** — use the asset's currently-set leverage; only change if
  user names a number ("5x", "10x").
- **Order type** — `Ioc` cross-book for "buy now" / no qualifier; `Gtc`
  resting limit only when user names a price; `FrontendMarket` for HIP-3
  stock perps.
- **Size** — USD notional → base size via `sz = usd / mid` from
  `/market/mids`; round to `szDecimals` from `/perp-metas`.

### Pre-flight (always)

Run the auto-unify rule, then:

```
required = (sz * px) / leverage     # perp / HIP-3
required = sz * px                  # spot buy

if unifiedFreeUSDC < required: walk the funding ladder
```

Spot sells: read the base token's `total` from `/account/spot`.

#### Funding ladder (when short)

1. **Bridge from another chain.** `GET /agent/wallets`; if any chain has
   USDC, bridge via `openfin-relay` into the Hyperliquid deposit address.
2. **Onramp** (if every wallet is empty). Show wallet addresses and
   suggest the user deposit, then bridge → trade.

Common failure mode: agent reads only `account.withdrawable`, sees `$0`,
suggests onramp, never checks spot or other wallets — which often have
the funds.

### Place — `POST /agent/trading/orders`

Body uses Hyperliquid's **single-letter** field names. Human-readable
ones (`coin` / `is_buy` / `sz` / `limit_px` / `order_type` /
`reduce_only`) appear only in GET responses; the **request side rejects
them**.

❌ **Wrong** — crashes with `Cannot read properties of undefined (reading 'limit')` because the validator reads `order.t.limit.tif`:

```json
{ "orders": [{ "coin": "xyz:NVDA", "is_buy": false, "sz": 0.052,
               "limit_px": 384.96,
               "order_type": { "limit": { "tif": "FrontendMarket" } },
               "reduce_only": false }],
  "grouping": "na" }
```

✅ **Right** — same trade, correct shape:

```json
{ "orders": [{ "a": 110012, "b": false, "p": "384.96", "s": "0.052",
               "r": false,
               "t": { "limit": { "tif": "FrontendMarket" } } }],
  "grouping": "na" }
```

Per-order fields:

- **`a`** — numeric asset index (formula below). Main-DEX BTC=0, ETH=1, …;
  HIP-3 e.g. `xyz:NVDA` = 110012. Never send a coin symbol.
- **`b`** — `true` = buy, `false` = sell.
- **`p`** — price; string is safest (`"65000"`). Always string for spot.
- **`s`** — size, **string** in base-asset units (`"0.01"`), pre-rounded
  to `szDecimals` (backend does NOT re-round size).
- **`r`** — bool, reduce-only.
- **`t`** — order type, exactly one branch:
  - `{ "limit": { "tif": "Gtc" | "Ioc" | "Alo" | "FrontendMarket" } }`
  - `{ "trigger": { "isMarket": bool, "triggerPx": "string-or-number",
       "tpsl": "tp" | "sl" } }` — for stop / take-profit.
- **`c`** — optional cloid (client order id) for idempotency.

Top-level `grouping`: `"na"` (default) | `"normalTpsl"` (group TP/SL with
a primary order) | `"positionTpsl"` (attach TP/SL to existing position).

### Asset index formula (HIP-3 included)

- Main-DEX perps: `a = index_in_meta` (BTC=0, ETH=1, …).
- HIP-3 perps: `a = 100000 + perp_dex_index * 10000 + index_in_meta`.
- Spot: `a = 10000 + spotInfo.index`.

Pull `index_in_meta` from `/market/perp-metas` per-DEX, `perp_dex_index`
from the `perpDexs` info call. Example: `xyz:NVDA` on
`perp_dex_index=1`, `index_in_meta=12` → `a = 110012`. Reference:
https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids

### HIP-3 perps (tokenized equities, etc.)

Same body as main DEX — same `a/b/p/s/r/t` fields. Only difference:
`a` is a large offset index. The backend handles builder fee + signing
internally; do not send a `builder` field. `FrontendMarket` is the
typical TIF for stock perps.

### Read

- **`GET /agent/trading/orders`** — Open orders (compact). `coin`, `side`
  (B/A), `limitPx`, `sz`, `oid`, `timestamp`, `orderType`.
- **`GET /agent/trading/orders/details`** — Full details: adds `origSz`,
  `triggerPx`, `triggerCondition`, `cloid`, `reduceOnly`.
- **`GET /agent/trading/orders/history`** — Filled / cancelled /
  rejected. Adds `statusMsg`, `filledSz`, `avgFillPx`, `cloid`.
- **`GET /agent/trading/orders/:oid/status`** — Single order status.

### Cancel / modify

- **`DELETE /agent/trading/orders`** body `{cancels: [{a, o}]}` —
  `a` = numeric asset index, `o` = order ID. Batch-cancellable.
- **`PUT /agent/trading/orders/:oid`** body `{order: {…}}` — Modify;
  same order shape as place.

### TWAP

- **`POST /agent/trading/twap`** — Place a TWAP that splits a large
  trade into market slices over a duration. Returns `twapId`.
- **`GET /agent/trading/twap/fills`** — Slice executions across all
  TWAPs. Returns fills with parent `twapId`.
- **`DELETE /agent/trading/twap/:twapId`** — Terminate; already-filled
  slices remain settled.

## Leverage & margin

- **`POST /agent/trading/leverage`** body `{asset, isCross, leverage}` —
  Set leverage + margin mode. `asset` = numeric index (same `a`).
  `isCross: true` = shared cross pool, `false` = isolated per-position.
  Symbol like `"xyz:NVDA"` returns `Failed to update leverage` — must
  be numeric.
- **`POST /agent/trading/margin`** body `{asset, isBuy, ntli}` — Add
  (`ntli > 0`, lowers liquidation risk) or remove margin from an
  isolated-margin position. Cross-margin positions don't accept this.

## Fills & funding

- **`GET /fills`** (`?aggregateByTime=`) — Latest fills. `coin`, side,
  `px`, `sz`, `feeUsdc`, `oid`, `cloid`, `crossed`, `liquidation`.
- **`GET /fills/by-time`** (`?startTime&endTime`) — Same, time-windowed (ms).
- **`GET /funding`** — Perp funding payment history. `usdc > 0` =
  received, `< 0` = paid.

---

## Sizing gotchas

- `s` is **base-asset** (BTC, ETH), not USD. BTC at $65k, `s="0.01"` =
  $650 notional.
- Tick size differs per asset — round `p` per-asset using
  `szDecimals` from `perp-metas`.
- Hyperliquid rejects prices outside the band (~5–10% from mid). For
  aggressive fills, cross the book but stay in-band.

---

## Don't

- Don't use `Gtc` for "buy now" — use `Ioc` cross-book.
- Don't put USD amounts in `s` — base-asset only.
- Don't trust `account.withdrawable` alone for free USDC — sum with
  `spot.USDC.total`.
- Don't reason about "perp USDC" vs "spot USDC" as separate balances,
  and don't surface the split in user replies (see Reply format).
- Don't suggest onramp before reading the unified balance + checking
  other wallets.
- Don't send a coin symbol as `a` (or as `asset` on `/leverage`,
  `/margin`, cancel) — must be the numeric index.

---

## MCP

Hyperliquid is a single dispatch tool: `hyperliquid` with an `action`
enum (`get_all_mids`, `get_market_metas`, `get_l2_book`,
`get_token_details`, `get_candle_snapshot`, `get_deposit_address`,
`get_account_summary` (+ `view: "perp"|"spot"|"all"`), `get_rate_limit`,
`get_open_orders` (+ `verbose`), `get_historical_orders`,
`get_order_status`, `place_order`, `cancel_order`, `modify_order`,
`place_twap_order`, `get_twap_fills`, `terminate_twap_order`,
`get_user_fills`, `get_user_funding_history`, `update_leverage`,
`update_isolated_margin`, `withdraw_to_arbitrum`, `get_user_abstraction`,
`set_user_abstraction`, `set_all_dexs_asset_ctxs_subscription`,
`get_all_dexs_asset_ctxs`, `get_ws_status`).
