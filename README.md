# Sir Trading AI Coding Skill

AI coding skill for [Sir Trading](https://app.sir.trading), a no-liquidation leveraged token protocol on MegaETH (chain 4326).

## What's Included

[`sir-trading.md`](sir-trading.md) covers:

- **APE/TEA minting & burning** — all 3 deposit methods (collateral, debt token, native ETH)
- **Quoting** — expected output before minting/burning
- **Vault discovery & creation** — enumerate vaults, check status, initialize new ones
- **MegaSIR staking** — stake, unstake, claim dividends (12 decimals!)
- **Fee auctions** — start auctions, bid, claim lots
- **Position introspection** — batch-read balances, unclaimed rewards, lock status
- **Foundry integration** — Solidity examples for on-chain composability

## Usage

### Claude Code / Cursor / Windsurf

Add the skill file to your project or reference it in your AI tool's context.

### Direct Reference

Point your AI coding assistant to the raw file:

```
https://raw.githubusercontent.com/SIR-trading/sir-trading-skill/main/sir-trading.md
```

## Resources

- [Sir Trading App](https://app.sir.trading)
- [Sir Trading Docs](https://docs.sir.trading)
- [Core Contracts](https://github.com/SIR-trading/Core)
- [Periphery Contracts](https://github.com/SIR-trading/Periphery)
