# SIR Trading AI Coding Skill

AI coding skill for [SIR Trading](https://app.sir.trading), a no-liquidation leveraged-token protocol. The same contracts are deployed on **Ethereum (1), HyperEVM (999), and MegaETH (4326)**, and this skill covers all three from one code path.

## What's Included

[`sir-trading.md`](sir-trading.md) covers:

- **APE/TEA minting & burning** — all three deposit methods (collateral, debt token, native gas token)
- **Quoting** — expected output before minting/burning
- **Vault discovery & creation** — subgraph and on-chain enumeration, status checks, initialize
- **SIR staking** — stake, unstake, claim native dividends (12 decimals!)
- **Reward claiming** — LP and contributor rewards (minted as SIR)
- **Fee auctions** — start, bid (native or wrapped), claim lots
- **Position introspection** — batch balances, unclaimed rewards, lock status
- **Per-chain notes** — DEX/oracle source, native token, SIR symbol, and the TEA lock period (MegaETH only)

The skill points at two always-fresh data sources so addresses and parameters never go stale:

- **`https://app.sir.trading/build-data.json`** — per-chain contract addresses + live system parameters + subgraph URL
- **The per-chain subgraph** — indexed vaults, positions, auctions, and metrics

## Usage

### Claude Code / Cursor / Windsurf

Add the skill file to your project or reference it in your AI tool's context.

### Direct Reference

Point your AI coding assistant at either mirror:

```
https://raw.githubusercontent.com/SIR-trading/sir-trading-skill/master/sir-trading.md
https://app.sir.trading/sir-trading.md
```

## Resources

- [SIR Trading App](https://app.sir.trading) · [Live data](https://app.sir.trading/build-data.json)
- [Docs](https://docs.sir.trading)
- [Core Contracts](https://github.com/SIR-trading/Core) · [Periphery Contracts](https://github.com/SIR-trading/Periphery)
