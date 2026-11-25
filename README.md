# openfinance-tech/skills

Agent skills for [OpenFinance](https://openfinance.tech): curated playbooks
that teach AI agents how to trade on Polymarket and Hyperliquid, bridge via
Relay, launch tokens, and use the user's OpenFinance wallets correctly.

Skills lead with the actual API routes (`/agent/…`) and WebSocket channels,
so they're useful whether you're integrating via direct HTTP/WS, through an
SDK, or via the [OpenFinance MCP server](https://github.com/openfinance-tech/mcp).

## Install

```bash
# Project-level (recommended: commit with your codebase)
npx skills add openfinance-tech/skills

# Or global
npx skills add openfinance-tech/skills -g

# Install to a specific agent only
npx skills add openfinance-tech/skills -a claude-code
```

Requires the CLI from [vercel-labs/skills](https://github.com/vercel-labs/skills).
Add `-y` to skip the prompts.

## What you need

- An OpenFinance account. Sign in at [openfinance.tech](https://openfinance.tech);
  your EVM and Solana wallets are created for you.
- An API key (it starts with `open_`). Open **External AI** in the sidebar and
  copy it. Keep it in the `OPENFINANCE_API_KEY` environment variable.

There's nothing to host. Your agent calls OpenFinance's hosted API, using the skills as instructions. Start with
`openfin-setup`, which checks the key before any other skill runs. Docs:
[docs.openfinance.tech](https://docs.openfinance.tech).

## Skills included

| Skill | Triggers on |
|---|---|
| [`openfin-setup`](./skills/openfin-setup) | First-time user, API key check, 401/412 auth errors |
| [`openfin-troubleshooting`](./skills/openfin-troubleshooting) | "Why is this failing", allowance errors, RPC issues, setup-incomplete errors |
| [`openfin-polymarket`](./skills/openfin-polymarket) | Markets, orderbooks, orders, positions/PnL, leaderboard, deposit / withdraw via bridge.polymarket.com |
| [`openfin-hyperliquid`](./skills/openfin-hyperliquid) | Perp/spot trading, leverage, TWAP, WS market data, unifiedAccount auto-upgrade |
| [`openfin-relay`](./skills/openfin-relay) | Cross-chain swaps and bridging between your own wallets, Solana routes |
| [`openfin-onchain`](./skills/openfin-onchain) | Token metadata, wallet portfolios, balances, USD prices, same-chain transfers |
| [`openfin-launchpad`](./skills/openfin-launchpad) | Solana token launchpad (Meteora DBC): launch, trade on the curve, migrate to DAMM v2, claim creator fees |
| [`openfin-onramp`](./skills/openfin-onramp) | Fiat → crypto via Moonpay (cards, global) or Onramp.money (UPI / IMPS, India) |

For AI agents: [AGENTS.md](./AGENTS.md) has the short version.

## Risk & audits

These skills sign real on-chain transactions on a user's wallet through
OpenFinance. Registry auditors flag the trading skills as **MEDIUM-risk** on
that basis, and that flag is correct:

| Auditor | Finding | What it means |
|---|---|---|
| Snyk `W009` | "Direct money access capability detected" (risk 1.00) | The skill drives OpenFinance's signing endpoints. Category-based rule: it fires on any signing skill. |
| Socket | `SUSPICIOUS` MEDIUM | "Purpose-aligned, no install-chain or malware indicators" but enables real-world asset movement. |

Neither finding identifies a malicious pattern, dependency hazard, or
hidden behavior. Both are honest signals that the skills can move funds,
which is the whole point.

What's already in place to keep that capability safe:

- **Each transactional skill ships an explicit `## Safety contract`**:
  read-only quote first, full disclosure of amounts/chains/fees, **explicit
  per-write user confirmation** in chat, no recipient/contract addresses
  pulled from untrusted content, re-quote on any parameter change. See
  [`SECURITY.md`](./SECURITY.md) for the repo-wide version.
- **Agents can't transfer, bridge or withdraw to anyone else.** The
  OpenFinance server refuses any send, bridge, swap or withdrawal whose
  destination isn't one of the caller's own wallets (EVM EOA + Solana
  via `get_wallet_addresses`, Polymarket deposit wallet via
  `get_deposit_wallet`). Sending to another person happens only on the
  Wallet page at openfinance.tech, with a code from the user's
  authenticator app.
- **No secrets, scripts, or binaries in this repo**: only markdown.
  Network calls happen from the agent and OpenFinance's servers, not
  from anything installed by `npx skills add`.

Found a way these skills could mislead an agent into a write without
confirmation? Open an issue.
