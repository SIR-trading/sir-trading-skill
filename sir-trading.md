# SIR Trading

AI coding skill for integrating [SIR Trading](https://app.sir.trading), a no-liquidation leveraged-token protocol. The same contracts are deployed on **Ethereum, HyperEVM, and MegaETH**. This skill lets an agent go long (mint APE), provide liquidity (mint TEA), close/reduce positions (burn), quote prices, create pairs (vaults), stake SIR, claim rewards, run fee auctions, discover pairs, and track portfolios — on any of the three chains from one code path.

> The protocol is identical on every chain. Almost everything in this skill is chain-agnostic; the only inputs that change per chain are **addresses, the DEX/oracle source, the native token, the SIR symbol, and whether TEA has a lock period**. Read [Chains](#chains) and [Per-Chain Notes](#per-chain-notes) and you can write one integration that works everywhere.

## Why SIR

SIR is a primitive for holding leverage long-term. Margin has liquidation and ongoing borrow fees. Leveraged tokens suffer volatility decay from rebalancing. Perps have liquidation and funding. Options have time decay and expiry. SIR removes those: **no liquidation, no funding while holding, and best-effort mitigation of volatility decay.** Fees are charged only on mint and burn. APE holders get a convex payoff (compounding gains up, bounded loss down). LPs (TEA holders) take the other side and collect the mint fees, like Uniswap LPing.

## When to use this skill

- Go long / open a leveraged position → **mint APE**
- Close or reduce a position → **burn APE**
- Provide liquidity / LP → **mint TEA**; withdraw → **burn TEA**
- Quote expected output before minting/burning
- Discover existing pairs (vaults) and read reserves/metrics
- Create a new pair (initialize a vault)
- Stake / unstake SIR; claim staking dividends (paid in the chain's native token)
- Claim LP rewards and contributor rewards (paid in SIR)
- Start fee auctions, bid, claim auction lots
- Build trading bots and portfolio trackers (read positions and unclaimed rewards)

## Architecture

- **Vault (called "Pair" in the app UI)** — a singleton contract holding every vault. A vault is identified by the struct `VaultParameters { address debtToken, address collateralToken, int8 leverageTier }`, and also by a sequential `vaultId` (`uint48`, starts at 1).
- **APE** — the leveraged long token. **A separate ERC-20 contract per vault.** Get its address from the subgraph (`vault.ape.id`) or `Assistant.getAddressAPE(vaultId)`. Decimals are per-vault (`ape.decimals`).
- **TEA** — the LP token. **A single ERC-1155 hosted by the Vault contract**, where the ERC-1155 `tokenId` **equals the `vaultId`**. TEA decimals **equal the collateral token's decimals** (not 18).
- **No liquidation.** APE has a convex payoff with bounded downside; APE holders cannot be liquidated.
- **Leverage** = `1 + 2^leverageTier`. Tiers run from `-4` (1.0625×) to `+2` (5×). Examples: tier `0` → 2×, `1` → 3×, `2` → 5×.
- **SIR has 12 decimals**, not 18, on every chain. Use `parseUnits(amount, 12)`, never `parseEther`.
- **Oracle** — a 30-minute TWAP from the chain's Uniswap-V3-style DEX (see [Chains](#chains)).
- **Assistant** is a read-only periphery contract for quotes, batch reads, and address lookups. The **Vault** executes state changes (`mint`, `burn`, `initialize`). The **SIR** contract handles staking, reward claiming, and fee auctions.

### Terminology (app UI label → contract term)

| App UI | Contract | Meaning |
|---|---|---|
| Pair | Vault | A leveraged token: collateral (base) vs debt (quote) at a fixed leverage |
| Go Long | Mint APE | Open a leveraged position |
| Close / Reduce | Burn APE | Exit a long, fully or partially |
| Provide Liquidity | Mint TEA | Deposit collateral as an LP |
| Withdraw | Burn TEA | Exit an LP position |
| New Pair | `initialize` | Create a new vault |
| Longs | APE positions | Ledger / Leaderboard tab |
| LP | TEA positions | Ledger / Leaderboard tab |
| Stake / Unstake | `stake` / `unstake` | SIR staking |
| Claim Rewards | `lperMint` / `claim` | Claim SIR (LP) or native dividends (stakers) |
| Ledger | User positions page | Open longs, LP, staking, closed positions |
| The Exchange | Leverage page | Going long |
| The Tea Room | Liquidity page | Providing liquidity |
| Auctions | Fee auctions | Bidding on accumulated fees |

## Live data sources (read these instead of hardcoding)

Addresses and system parameters are deployment state, not constants. Two canonical, always-fresh sources back the production app — prefer them over the static tables below:

1. **`https://app.sir.trading/build-data.json`** — per chain: every contract address, the live system parameters (`baseFee`, `lpFee`, `lpLockTime`, `mintingStopped`, `cumulativeTax`), `timestampIssuanceStart`, and the `subgraphUrl`. This is generated at each deploy by resolving everything on-chain from the Assistant address.

2. **The subgraph** (URL per chain in `build-data.json`) — indexed vaults, positions, auctions, and metrics. The efficient path for discovery and portfolio reads; on-chain enumeration is the trustless fallback.

Everything bootstraps from **one address per chain — the Assistant.** `Assistant.VAULT()` → `Vault.SIR()` / `.ORACLE()` → `SIR.CONTRIBUTORS()`. So if you only trust on-chain reads, you need just the Assistant address; derive the rest.

```ts
// Always-fresh addresses + params, no redeploy drift
const buildData = await fetch("https://app.sir.trading/build-data.json").then(r => r.json());
const eth = buildData.chains["1"];
// eth.contractAddresses.{vault,assistant,sir,oracle,systemControl,...}
// eth.systemParams.{baseFee,lpFee,mintingStopped,cumulativeTax,lpLockTime?}
// eth.subgraphUrl
```

## Chains

| | Ethereum | HyperEVM | MegaETH |
|---|---|---|---|
| Chain ID | `1` | `999` | `4326` |
| Native token | ETH | **HYPE** | ETH |
| Wrapped native | WETH `0xC02a…756Cc2` | **WHYPE `0x5555…5555`** | WETH `0x4200…0006` |
| SIR symbol | **SIR** | **HyperSIR** | **MegaSIR** |
| SIR decimals | 12 | 12 | 12 |
| DEX / oracle source | Uniswap v3 | **HyperSwap** | **Kumbaya** |
| TEA lock period | **none** | **none** | **optional, ≤90 days** (cuts mint fee) |
| Auction min bid increment | **1%** | 5% | 5% |
| Oracle TWAP window | 30 min | 30 min | 30 min |
| Default stablecoin (quote) | USDT | USD₮0 | USDM |
| Explorer | etherscan.io | hyperevmscan.io | mega.etherscan.io |

`leverageTier`, decimals (SIR=12), the leverage formula, and every ABI are the same on all three. The behavioral forks are small but real: the **TEA lock period** (MegaETH only) and the **auction min bid increment** (1% on Ethereum vs 5% elsewhere). See [Per-Chain Notes](#per-chain-notes).

### Contract addresses (snapshot — verify against `build-data.json`)

| Contract | Ethereum (1) | HyperEVM (999) | MegaETH (4326) |
|---|---|---|---|
| Assistant | `0xff14f91285580AEd3733c0B1F3C8b6d04804c5ec` | `0x7d987b986FbA5e0A4247649A2334Bb2D4029656c` | `0xB91AE2c8365FD45030abA84a4666C4dB074E53E7` |
| Vault | `0x7Dad75dD36dE234C937C105e652B6E50d68b0309` | `0x4a35e7448Dad9cAc6B3e529050B5a6Ee56A0eDF0` | `0x8d694D1b369BdE5B274Ad643fEdD74f836E88543` |
| SIR | `0x4Da4fb565Dcd5D5C5dB495205c109bA983A8ABa2` | `0xA06D0c5a8ADb7134903CA13D1FC0641731E2B766` | `0x9367A0c482703d8d9bda995B03f8E71056a72500` |
| Oracle | `0xeD89aF5E62965C45956A0125a5d078218228497A` | `0x2Ab530127a40a832B3e9AD2F0eC6Cdfee17542E0` | `0x4edF071a7dEe52fBE663DF7873994725ba91Cdc7` |
| Wrapped native | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | `0x5555555555555555555555555555555555555555` | `0x4200000000000000000000000000000000000006` |
| DEX SwapRouter | `0xE592427A0AEce92De3Edee1F18E0157C05861564` | `0x6D99e7f6747AF2cDbB5164b6DD50e40D4fDe1e77` | `0xE5BbEF8De2DB447a7432A47EBa58924d94eE470e` |
| DEX QuoterV2 | `0x61fFE014bA17989E743c5F6cB21bF9697530B21e` | `0x03A918028f22D9E1473B7959C927AD7425A45C7C` | `0x1F1a8dC7E138C34b503Ca080962aC10B75384a27` |
| DEX Factory | `0x1F98431c8aD98523631AE4a59f267346ea31F984` | `0xB1c0fa0B789320044A6F623cFe5eBda9562602E3` | `0x68b34591f662508076927803c567Cc8006988a09` |

### System parameters (snapshot — read live from `build-data.json`)

Identical across chains today, but governance-settable and read live; never hardcode for fee math:

| Param | Value | Meaning |
|---|---|---|
| `baseFee` | `1000` (= `0.1` in build-data) | APE fee coefficient (see [Fees](#fee-structure)) |
| `lpFee` | `515` bps (= `0.0515`) | TEA mint fee coefficient |
| `lpLockTime` | `7776000`s (90d) **MegaETH only** | TEA max lock duration; absent ⇒ no lock |
| `mintingStopped` | `false` | If `true`, minting is disabled protocol-wide — check before minting |
| `cumulativeTax` | 493 / 347 / 543 | Vault tax; sets the staker share of fees |

## ABIs

Minimal ABIs covering everything in this skill. Full ABIs live in the [Core](https://github.com/SIR-trading/Core) and [Periphery](https://github.com/SIR-trading/Periphery) repos.

### Vault ABI

```typescript
export const vaultAbi = [
  // --- state-changing ---
  { type: "function", name: "mint", stateMutability: "payable",
    inputs: [
      { name: "isAPE", type: "bool" },
      { name: "vaultParams", type: "tuple", components: [
        { name: "debtToken", type: "address" },
        { name: "collateralToken", type: "address" },
        { name: "leverageTier", type: "int8" }] },
      { name: "amountToDeposit", type: "uint256" },       // collateral if min==0, else debt token
      { name: "collateralToDepositMin", type: "uint144" },// > 0 signals a debt-token deposit (swap)
      { name: "deadline", type: "uint40" },               // unix secs; 0 = no deadline
      { name: "portionLockTime", type: "uint8" }],        // TEA only; 0 = full fee, 255 = zero fee (MegaETH)
    outputs: [{ name: "amount", type: "uint256" }] },
  { type: "function", name: "burn", stateMutability: "nonpayable",
    inputs: [
      { name: "isAPE", type: "bool" },
      { name: "vaultParams", type: "tuple", components: [
        { name: "debtToken", type: "address" },
        { name: "collateralToken", type: "address" },
        { name: "leverageTier", type: "int8" }] },
      { name: "amount", type: "uint256" },
      { name: "deadline", type: "uint40" }],
    outputs: [{ name: "", type: "uint144" }] },
  { type: "function", name: "initialize", stateMutability: "nonpayable",
    inputs: [{ name: "vaultParams", type: "tuple", components: [
      { name: "debtToken", type: "address" },
      { name: "collateralToken", type: "address" },
      { name: "leverageTier", type: "int8" }] }],
    outputs: [] },
  // --- views ---
  { type: "function", name: "getReserves", stateMutability: "view",
    inputs: [{ name: "vaultParams", type: "tuple", components: [
      { name: "debtToken", type: "address" },
      { name: "collateralToken", type: "address" },
      { name: "leverageTier", type: "int8" }] }],
    outputs: [{ name: "", type: "tuple", components: [
      { name: "reserveApes", type: "uint144" },
      { name: "reserveLPers", type: "uint144" },
      { name: "tickPriceX42", type: "int64" }] }] },
  { type: "function", name: "vaultStates", stateMutability: "view",
    inputs: [{ name: "vaultParams", type: "tuple", components: [
      { name: "debtToken", type: "address" },
      { name: "collateralToken", type: "address" },
      { name: "leverageTier", type: "int8" }] }],
    outputs: [{ name: "", type: "tuple", components: [
      { name: "reserve", type: "uint144" },
      { name: "tickPriceSatX42", type: "int64" },
      { name: "vaultId", type: "uint48" }] }] },
  { type: "function", name: "paramsById", stateMutability: "view",
    inputs: [{ name: "vaultId", type: "uint48" }],
    outputs: [{ name: "", type: "tuple", components: [
      { name: "debtToken", type: "address" },
      { name: "collateralToken", type: "address" },
      { name: "leverageTier", type: "int8" }] }] },
  { type: "function", name: "numberOfVaults", stateMutability: "view", inputs: [],
    outputs: [{ name: "", type: "uint48" }] },
  // --- TEA (ERC-1155 on the Vault; tokenId == vaultId) ---
  { type: "function", name: "balanceOf", stateMutability: "view",
    inputs: [{ name: "account", type: "address" }, { name: "vaultId", type: "uint256" }],
    outputs: [{ name: "", type: "uint256" }] },
  { type: "function", name: "totalSupply", stateMutability: "view",
    inputs: [{ name: "vaultId", type: "uint256" }], outputs: [{ name: "", type: "uint256" }] },
  { type: "function", name: "safeTransferFrom", stateMutability: "nonpayable",
    inputs: [{ name: "from", type: "address" }, { name: "to", type: "address" },
      { name: "vaultId", type: "uint256" }, { name: "amount", type: "uint256" }, { name: "data", type: "bytes" }],
    outputs: [] },
  { type: "function", name: "setApprovalForAll", stateMutability: "nonpayable",
    inputs: [{ name: "operator", type: "address" }, { name: "approved", type: "bool" }], outputs: [] },
  { type: "function", name: "unclaimedRewards", stateMutability: "view",
    inputs: [{ name: "vaultId", type: "uint256" }, { name: "lper", type: "address" }],
    outputs: [{ name: "", type: "uint80" }] },
  { type: "function", name: "lockEnd", stateMutability: "view",
    inputs: [{ name: "account", type: "address" }, { name: "vaultId", type: "uint256" }],
    outputs: [{ name: "", type: "uint40" }] },
] as const;
```

### SIR ABI (token + staking + rewards + auctions)

```typescript
export const sirAbi = [
  // --- staking ---
  { type: "function", name: "stake", stateMutability: "nonpayable",
    inputs: [{ name: "amount", type: "uint80" }], outputs: [] },
  { type: "function", name: "unstake", stateMutability: "nonpayable",
    inputs: [{ name: "amount", type: "uint80" }], outputs: [] },
  { type: "function", name: "claim", stateMutability: "nonpayable",
    inputs: [], outputs: [{ name: "dividends_", type: "uint96" }] },
  { type: "function", name: "unstakeAndClaim", stateMutability: "nonpayable",
    inputs: [{ name: "amount", type: "uint80" }], outputs: [{ name: "dividends_", type: "uint96" }] },
  { type: "function", name: "stakeOf", stateMutability: "view",
    inputs: [{ name: "staker", type: "address" }],
    outputs: [{ name: "unlockedStake", type: "uint80" }, { name: "lockedStake", type: "uint80" }] },
  { type: "function", name: "unclaimedDividends", stateMutability: "view",
    inputs: [{ name: "staker", type: "address" }], outputs: [{ name: "", type: "uint96" }] },
  // --- reward claiming (mints SIR) ---
  { type: "function", name: "lperMint", stateMutability: "nonpayable",
    inputs: [{ name: "vaultId", type: "uint256" }], outputs: [{ name: "rewards", type: "uint80" }] },
  { type: "function", name: "lperMintAndStake", stateMutability: "nonpayable",
    inputs: [{ name: "vaultId", type: "uint256" }], outputs: [{ name: "rewards", type: "uint80" }] },
  { type: "function", name: "contributorMint", stateMutability: "nonpayable",
    inputs: [], outputs: [{ name: "rewards", type: "uint80" }] },
  { type: "function", name: "contributorMintAndStake", stateMutability: "nonpayable",
    inputs: [], outputs: [{ name: "rewards", type: "uint80" }] },
  { type: "function", name: "contributorUnclaimedSIR", stateMutability: "view",
    inputs: [{ name: "contributor", type: "address" }], outputs: [{ name: "", type: "uint80" }] },
  // --- fee auctions ---
  { type: "function", name: "collectFeesAndStartAuction", stateMutability: "nonpayable",
    inputs: [{ name: "token", type: "address" }], outputs: [{ name: "totalFees", type: "uint256" }] },
  { type: "function", name: "bid", stateMutability: "payable",
    inputs: [{ name: "token", type: "address" }, { name: "amount", type: "uint96" }], outputs: [] },
  { type: "function", name: "getAuctionLot", stateMutability: "nonpayable",
    inputs: [{ name: "token", type: "address" }, { name: "beneficiary", type: "address" }], outputs: [] },
  { type: "function", name: "auctions", stateMutability: "view",
    inputs: [{ name: "token", type: "address" }],
    outputs: [{ name: "", type: "tuple", components: [
      { name: "bidder", type: "address" }, { name: "bid", type: "uint96" }, { name: "startTime", type: "uint40" }] }] },
  // --- ERC-20 (12 decimals) ---
  { type: "function", name: "balanceOf", stateMutability: "view",
    inputs: [{ name: "account", type: "address" }], outputs: [{ name: "", type: "uint256" }] },
  { type: "function", name: "approve", stateMutability: "nonpayable",
    inputs: [{ name: "spender", type: "address" }, { name: "amount", type: "uint256" }],
    outputs: [{ name: "", type: "bool" }] },
  { type: "function", name: "transfer", stateMutability: "nonpayable",
    inputs: [{ name: "to", type: "address" }, { name: "amount", type: "uint256" }],
    outputs: [{ name: "", type: "bool" }] },
  { type: "function", name: "decimals", stateMutability: "view", inputs: [], outputs: [{ name: "", type: "uint8" }] },
] as const;
```

### Assistant ABI (quotes + batch reads)

```typescript
export const assistantAbi = [
  { type: "function", name: "quoteMint", stateMutability: "view",
    inputs: [{ name: "isAPE", type: "bool" },
      { name: "vaultParams", type: "tuple", components: [
        { name: "debtToken", type: "address" }, { name: "collateralToken", type: "address" }, { name: "leverageTier", type: "int8" }] },
      { name: "amountCollateral", type: "uint144" }],
    outputs: [{ name: "amountTokens", type: "uint256" }] },
  { type: "function", name: "quoteMintWithDebtToken", stateMutability: "view",
    inputs: [{ name: "isAPE", type: "bool" },
      { name: "vaultParams", type: "tuple", components: [
        { name: "debtToken", type: "address" }, { name: "collateralToken", type: "address" }, { name: "leverageTier", type: "int8" }] },
      { name: "amountDebtToken", type: "uint256" }],
    outputs: [{ name: "amountTokens", type: "uint256" }, { name: "amountCollateral", type: "uint256" }, { name: "amountCollateralIdeal", type: "uint256" }] },
  { type: "function", name: "quoteBurn", stateMutability: "view",
    inputs: [{ name: "isAPE", type: "bool" },
      { name: "vaultParams", type: "tuple", components: [
        { name: "debtToken", type: "address" }, { name: "collateralToken", type: "address" }, { name: "leverageTier", type: "int8" }] },
      { name: "amountTokens", type: "uint256" }],
    outputs: [{ name: "amountCollateral", type: "uint144" }, { name: "amountDebtToken", type: "uint256" }] },
  { type: "function", name: "getReserves", stateMutability: "view",
    inputs: [{ name: "vaultIds", type: "uint48[]" }],
    outputs: [{ name: "reserves", type: "tuple[]", components: [
      { name: "reserveApes", type: "uint144" }, { name: "reserveLPers", type: "uint144" }, { name: "tickPriceX42", type: "int64" }] }] },
  { type: "function", name: "getUserBalances", stateMutability: "view",
    inputs: [{ name: "user", type: "address" }, { name: "offset", type: "uint256" }, { name: "numVaults", type: "uint256" }],
    outputs: [{ name: "apeBalances", type: "uint256[]" }, { name: "teaBalances", type: "uint256[]" }, { name: "unclaimedSirRewards", type: "uint80[]" }] },
  { type: "function", name: "getVaultStatus", stateMutability: "view",
    inputs: [{ name: "vaultParams", type: "tuple", components: [
      { name: "debtToken", type: "address" }, { name: "collateralToken", type: "address" }, { name: "leverageTier", type: "int8" }] }],
    outputs: [{ name: "", type: "uint8" }] },  // 0=Invalid 1=NoPool 2=CanBeCreated 3=AlreadyExists
  { type: "function", name: "getAddressAPE", stateMutability: "view",
    inputs: [{ name: "vaultId", type: "uint48" }], outputs: [{ name: "", type: "address" }] },
] as const;
```

## Setup (one config, any chain)

```typescript
import { createPublicClient, createWalletClient, http, defineChain, parseUnits } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains";
import { vaultAbi, sirAbi, assistantAbi } from "./abis";

// Snapshot config — for production, hydrate addresses/params from
// https://app.sir.trading/build-data.json so you never drift on redeploys.
export const CHAINS = {
  1: {
    chain: mainnet,
    rpc: "https://eth.llamarpc.com", // use your own RPC
    assistant: "0xff14f91285580AEd3733c0B1F3C8b6d04804c5ec",
    vault: "0x7Dad75dD36dE234C937C105e652B6E50d68b0309",
    sir: "0x4Da4fb565Dcd5D5C5dB495205c109bA983A8ABa2",
    wrappedNative: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
    sirSymbol: "SIR",
    hasLock: false,
    minBidIncrease: 0.01, // auction min bid increment — 1% on Ethereum
  },
  999: {
    chain: defineChain({ id: 999, name: "HyperEVM",
      nativeCurrency: { name: "HYPE", symbol: "HYPE", decimals: 18 },
      rpcUrls: { default: { http: ["https://rpc.hyperliquid.xyz/evm"] } } }),
    rpc: "https://rpc.hyperliquid.xyz/evm",
    assistant: "0x7d987b986FbA5e0A4247649A2334Bb2D4029656c",
    vault: "0x4a35e7448Dad9cAc6B3e529050B5a6Ee56A0eDF0",
    sir: "0xA06D0c5a8ADb7134903CA13D1FC0641731E2B766",
    wrappedNative: "0x5555555555555555555555555555555555555555",
    sirSymbol: "HyperSIR",
    hasLock: false,
    minBidIncrease: 0.05, // auction min bid increment — 5%
  },
  4326: {
    chain: defineChain({ id: 4326, name: "MegaETH",
      nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
      rpcUrls: { default: { http: ["https://mainnet.megaeth.com/rpc"] } } }),
    rpc: "https://mainnet.megaeth.com/rpc",
    assistant: "0xB91AE2c8365FD45030abA84a4666C4dB074E53E7",
    vault: "0x8d694D1b369BdE5B274Ad643fEdD74f836E88543",
    sir: "0x9367A0c482703d8d9bda995B03f8E71056a72500",
    wrappedNative: "0x4200000000000000000000000000000000000006",
    sirSymbol: "MegaSIR",
    hasLock: true,
    minBidIncrease: 0.05, // auction min bid increment — 5%
  },
} as const;

export function clients(chainId: keyof typeof CHAINS, privateKey?: `0x${string}`) {
  const c = CHAINS[chainId];
  const publicClient = createPublicClient({ chain: c.chain, transport: http(c.rpc) });
  const walletClient = privateKey
    ? createWalletClient({ account: privateKeyToAccount(privateKey), chain: c.chain, transport: http(c.rpc) })
    : undefined;
  return { c, publicClient, walletClient };
}

const erc20Abi = [
  { type: "function", name: "approve", stateMutability: "nonpayable",
    inputs: [{ name: "spender", type: "address" }, { name: "amount", type: "uint256" }],
    outputs: [{ name: "", type: "bool" }] },
  { type: "function", name: "allowance", stateMutability: "view",
    inputs: [{ name: "owner", type: "address" }, { name: "spender", type: "address" }],
    outputs: [{ name: "", type: "uint256" }] },
  { type: "function", name: "balanceOf", stateMutability: "view",
    inputs: [{ name: "account", type: "address" }], outputs: [{ name: "", type: "uint256" }] },
  { type: "function", name: "decimals", stateMutability: "view", inputs: [], outputs: [{ name: "", type: "uint8" }] },
] as const;
```

> **Defaults to verify:** RPC endpoints above are public examples — supply your own. Always re-read addresses from `build-data.json` in production.

## Default stack decisions

1. **Quote with Assistant, execute with Vault.** Assistant is read-only (`quoteMint`, `quoteBurn`, `getUserBalances`, …). Vault does `mint` / `burn` / `initialize`. SIR does staking, claiming, and auctions.
2. **SIR is 12 decimals.** `parseUnits(amount, 12)`.
3. **`VaultParameters` is a struct tuple** `{ debtToken, collateralToken, leverageTier }` — it identifies a vault. The sequential `vaultId` identifies the same vault for APE/TEA/rewards.
4. **APE address is per-vault** (`Assistant.getAddressAPE(vaultId)` or subgraph `vault.ape.id`). **TEA is the Vault's ERC-1155** with `tokenId == vaultId` and **collateral decimals**.

## Discover pairs (vaults)

### Via subgraph (preferred — one call gives APE addresses, reserves, leverage, volatility)

`vault.id` is the **hex** form of the vault ID (e.g. `"0x13"` = vault 19). Live vaults have `exists: true`.

```typescript
const SUBGRAPH = (await fetch("https://app.sir.trading/build-data.json").then(r => r.json()))
  .chains[String(chainId)].subgraphUrl;

const query = `{
  vaults(first: 100, orderBy: totalValueUsd, orderDirection: desc) {
    id exists leverageTier reserveApes reserveLPers totalValueUsd volatilityAnnual
    collateralToken { id symbol decimals }
    debtToken { id symbol decimals }
    ape { id symbol decimals }
  }
}`;
const { vaults } = await fetch(SUBGRAPH, {
  method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ query }),
}).then(r => r.json()).then(j => j.data);
// vaultId (decimal) = parseInt(vault.id, 16)
// vaultParams = { debtToken: debtToken.id, collateralToken: collateralToken.id, leverageTier }
```

### Via on-chain enumeration (trustless fallback)

```typescript
const { c, publicClient } = clients(chainId);
const n = await publicClient.readContract({ address: c.vault, abi: vaultAbi, functionName: "numberOfVaults" });
for (let id = 1n; id <= n; id++) {
  const p = await publicClient.readContract({ address: c.vault, abi: vaultAbi, functionName: "paramsById", args: [id] });
  // p.debtToken, p.collateralToken, p.leverageTier
}
// Batch reserves for many vaults at once:
const reserves = await publicClient.readContract({
  address: c.assistant, abi: assistantAbi, functionName: "getReserves",
  args: [[1, 2, 3]], // uint48[]
});
```

## Quote before acting

```typescript
const { c, publicClient } = clients(chainId);
const vaultParams = { debtToken, collateralToken, leverageTier };

// Mint with collateral
const apeOut = await publicClient.readContract({
  address: c.assistant, abi: assistantAbi, functionName: "quoteMint",
  args: [true, vaultParams, parseUnits("100", collateralDecimals)] });

// Mint with debt token (includes the DEX swap quote)
const [tokensOut, collateralAfterSwap, collateralIdeal] = await publicClient.readContract({
  address: c.assistant, abi: assistantAbi, functionName: "quoteMintWithDebtToken",
  args: [true, vaultParams, parseUnits("1", debtDecimals)] });

// Burn (isAPE=false for TEA). amountCollateral & amountDebtToken already net of the burn fee.
const [collateralOut, debtOut] = await publicClient.readContract({
  address: c.assistant, abi: assistantAbi, functionName: "quoteBurn",
  args: [true, vaultParams, apeAmount] });
```

## Mint APE (go long)

`mint(isAPE, vaultParams, amountToDeposit, collateralToDepositMin, deadline, portionLockTime)`. The deposit token is inferred: **`collateralToDepositMin == 0` ⇒ depositing collateral; `> 0` ⇒ depositing the debt token, which the Vault swaps to collateral on the chain's DEX** (set the min from a quote for slippage protection). `portionLockTime` is ignored for APE.

```typescript
const { c, publicClient, walletClient } = clients(chainId, pk);
const deadline = BigInt(nowSecs + 600); // pass `nowSecs` in; or 0n for no deadline

// (A) Deposit collateral directly
await walletClient.writeContract({ address: collateralToken, abi: erc20Abi, functionName: "approve", args: [c.vault, amount] });
await walletClient.writeContract({ address: c.vault, abi: vaultAbi, functionName: "mint",
  args: [true, vaultParams, amount, 0n, deadline, 0] });

// (B) Deposit the debt token (Vault swaps debt → collateral)
const [, quotedCollateral] = await publicClient.readContract({ address: c.assistant, abi: assistantAbi,
  functionName: "quoteMintWithDebtToken", args: [true, vaultParams, debtAmount] });
const minCollateral = (quotedCollateral * 99n) / 100n; // 1% slippage
await walletClient.writeContract({ address: debtToken, abi: erc20Abi, functionName: "approve", args: [c.vault, debtAmount] });
await walletClient.writeContract({ address: c.vault, abi: vaultAbi, functionName: "mint",
  args: [true, vaultParams, debtAmount, minCollateral, deadline, 0] });

// (C) Native gas token as collateral (vault collateral is the wrapped native, e.g. WETH/WHYPE)
await walletClient.writeContract({ address: c.vault, abi: vaultAbi, functionName: "mint",
  args: [true, vaultParams, 0n, 0n, deadline, 0], value: parseUnits("1", 18) });
```

> **Native-deposit guard:** when sending native value **and** the native maps to the vault's *debt* token (so a swap is required), `collateralToDepositMin` **must be `> 0`** — otherwise the contract checks `collateralToken == WETH` instead of `debtToken == WETH` and reverts `NotAWETHVault`. When the native is the *collateral*, `min = 0n` is correct.

## Mint TEA (provide liquidity)

Same call with `isAPE=false`. **`portionLockTime` only matters where the chain has a lock period (`lpLockTime > 0` — MegaETH today).** There, `0` = full fee / no lock, `255` = zero fee / maximum lock (90 days), and intermediate values trade fee for lock time. On Ethereum and HyperEVM there is no lock: pass `0`, the fee is fixed, and TEA is withdrawable immediately.

```typescript
// Collateral deposit; ~half lock for ~half fee reduction on MegaETH (portionLockTime=128)
await walletClient.writeContract({ address: c.vault, abi: vaultAbi, functionName: "mint",
  args: [false, vaultParams, amount, 0n, deadline, c.hasLock ? 128 : 0] });

// Debt-token and native variants mirror the APE flows above with isAPE=false.
```

## Burn (close / withdraw)

Returns collateral to the caller. For TEA on a lock chain, check `lockEnd` first.

```typescript
// Burn APE
await walletClient.writeContract({ address: c.vault, abi: vaultAbi, functionName: "burn",
  args: [true, vaultParams, apeAmount, deadline] });

// Burn TEA — on MegaETH, ensure it's unlocked
if (c.hasLock) {
  const lockEnd = await publicClient.readContract({ address: c.vault, abi: vaultAbi,
    functionName: "lockEnd", args: [account, vaultId] });
  if (lockEnd > BigInt(nowSecs)) throw new Error(`TEA locked until ${new Date(Number(lockEnd) * 1000)}`);
}
await walletClient.writeContract({ address: c.vault, abi: vaultAbi, functionName: "burn",
  args: [false, vaultParams, teaAmount, deadline] });
```

## Create a new pair (initialize a vault)

Requires a DEX pool to exist for the collateral/debt pair. `getVaultStatus`: `0`=Invalid, `1`=NoPool, `2`=CanBeCreated, `3`=AlreadyExists.

```typescript
const status = await publicClient.readContract({ address: c.assistant, abi: assistantAbi,
  functionName: "getVaultStatus", args: [vaultParams] });
if (status !== 2) throw new Error(`Cannot create (status ${status})`);
await walletClient.writeContract({ address: c.vault, abi: vaultAbi, functionName: "initialize", args: [vaultParams] });
```

> **Pick the leverage tier for the pair's volatility.** `maxTier = floor(-log2(volatility) - 1)` (volatility = annualized, from the subgraph's `volatilityAnnual` or a 30-day EWMA). Too high a tier for the pair's volatility makes the vault saturate frequently — bad for both traders and LPs.

## Read user positions

```typescript
// Batch APE + TEA + unclaimed SIR across vaults [offset+1 .. offset+numVaults]
const n = await publicClient.readContract({ address: c.vault, abi: vaultAbi, functionName: "numberOfVaults" });
const [apeBalances, teaBalances, unclaimedSir] = await publicClient.readContract({
  address: c.assistant, abi: assistantAbi, functionName: "getUserBalances",
  args: [user, 0n, BigInt(n)] });
```

Or via subgraph (`apePositions` / `teaPositions` by `user`) for cost basis (`collateralTotal`, `dollarTotal`, `debtTokenTotal`) and `lockEnd`.

## Stake / unstake / claim SIR

Staking moves your own SIR balance into staked state inside the SIR contract — **no `approve` needed.** Locked stake unlocks on a 30-day half-life (50% at 30d, 75% at 60d, …); only the unlocked portion can be unstaked. `claim` pays accrued dividends in the **chain's native token** (ETH / HYPE).

```typescript
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "stake", args: [parseUnits("100", 12)] });

const [unlocked, locked] = await publicClient.readContract({ address: c.sir, abi: sirAbi, functionName: "stakeOf", args: [account] });
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "unstake", args: [unlocked] }); // <= unlocked
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "claim" });                       // native dividends
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "unstakeAndClaim", args: [unlocked] });
```

## Claim rewards (LP + contributor)

LP and contributor rewards are minted as SIR. `*AndStake` mints and stakes in one tx.

```typescript
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "lperMint", args: [vaultId] });
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "lperMintAndStake", args: [vaultId] });
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "contributorMint" });
```

## Fee auctions

Protocol fees accumulate per token and are auctioned for the chain's dividend token (WETH / WHYPE / WETH). Bids are placed in the **native** token (ETH / HYPE); the winning bid becomes staker dividends and the winner collects the fee lot.

- **`bid` is `payable` on every chain.** Send native value as `msg.value` and the contract wraps it to the chain's WETH/WHYPE automatically; **or** send `msg.value: 0` after approving WETH/WHYPE to the SIR contract and it pulls `amount` via `transferFrom`.
- A new bid must beat the current one by the chain's **minimum increment — 1% on Ethereum, 5% on HyperEVM and MegaETH** — strictly (a bid exactly at the increment is rejected; e.g. on a 5% chain `100*amount > 105*currentBid`). If you are already the top bidder, an additional bid **adds** to your standing bid.
- Auctions last `AUCTION_DURATION` = **24h**, with `AUCTION_COOLDOWN` = **247h (~10.3 days)** before the same token can be auctioned again.

```typescript
const auction = await publicClient.readContract({ address: c.sir, abi: sirAbi, functionName: "auctions", args: [token] });
// Min increment is per-chain: 1% (Ethereum) or 5% (HyperEVM, MegaETH)
const incBps = 100n + BigInt(Math.round(c.minBidIncrease * 100)); // 101n or 105n
const bidAmount = (auction.bid * incBps) / 100n + 1n;

// Native bid (auto-wrapped)
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "bid", args: [token, bidAmount], value: bidAmount });

// Start an auction for a token's accrued fees
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "collectFeesAndStartAuction", args: [token] });

// Winner collects the lot after the auction ends
await walletClient.writeContract({ address: c.sir, abi: sirAbi, functionName: "getAuctionLot", args: [token, beneficiary] });
```

## Fee structure

System parameters are read live (`build-data.json` / SystemControl); current defaults are `baseFee = 0.1` and `lpFee = 0.0515` (the decimal forms; on-chain integers are 1000 and 515).

**APE (leverage) fee.** A fee is charged on **both** mint and burn, and the protocol's headline fee is the combined round trip. Each side multiplies the collateral by `1/(1+(ℓ−1)·baseFee)`, where `ℓ = 1 + 2^leverageTier`:

- **round-trip fee (the quoted fee — mint *and* burn) = `1 − 1/(1+(ℓ−1)·baseFee)²`**
- per-side fee (a single mint or single burn, what `quoteMint`/`quoteBurn` reflect) = `1 − 1/(1+(ℓ−1)·baseFee)`

With `baseFee = 0.1`:

| Leverage | tier | per side | round trip |
|---|:---:|:---:|:---:|
| 1.25× | −2 | 2.4% | 4.8% |
| 1.5× | −1 | 4.8% | 9.3% |
| 2× | 0 | 9.1% | 17% |
| 3× | 1 | 16.7% | 31% |
| 5× | 2 | 28.6% | 49% |

`quoteMint` / `quoteBurn` / `priceOfAPE` already net the relevant side, so you rarely compute this yourself. Recipients: LPers + stakers (staker share set by `cumulativeTax`).

**Other fees:**

| Fee | Value | Recipient |
|---|---|---|
| TEA mint | `1 − 1/(1+lpFee)`, `lpFee=0.0515` ⇒ **~4.9%**. Reducible toward 0 with `portionLockTime` where `lpLockTime>0` (**MegaETH only**). | Protocol-owned liquidity |
| TEA burn | 0% | — |
| Staker dividends | Distributed via fee auctions; **at most 50% of vault fees** go to stakers (the rest to LPs; split set by `cumulativeTax`). | SIR stakers, in the chain's **native token** |

## Per-chain notes

- **Ethereum (1):** Uniswap v3 oracle/swaps (30-min TWAP, 5-min update cadence). No TEA lock (`portionLockTime` is a no-op; pass `0`). Native ETH, SIR symbol `SIR`. **Auction min bid increment: 1%.** Standard L1 gas/mempool.
- **HyperEVM (999):** HyperSwap oracle/swaps (30-min TWAP, 1-min update cadence). No TEA lock. **Native token is HYPE; wrapped native is WHYPE `0x5555…5555`** — native deposits and staker dividends are in HYPE. SIR symbol `HyperSIR`. **Auction min bid increment: 5%.** Public RPC `https://rpc.hyperliquid.xyz/evm`.
- **MegaETH (4326):** Kumbaya oracle/swaps (30-min TWAP, 1-min update cadence). **TEA minting can optionally lock the position for up to 90 days to reduce the mint fee** — `portionLockTime` trades lock time for a lower fee (`0` = no lock / full fee → `255` = max 90-day lock / ~0 fee). It is never mandatory. While locked, `burn` reverts `TEALocked` (`lockEnd` in the future) and TEA transfers revert `TransferToLowerLockEnd` if the sender's lock end exceeds the recipient's. Native ETH, SIR symbol `MegaSIR`. **Auction min bid increment: 5%.** MegaETH-specific extras: `eth_sendRawTransactionSync` (EIP-7966) for sub-10ms receipts; sequencer-ordered txs (no public mempool); first-touch storage writes cost more.

## Safety notes for agents

- **Check `mintingStopped`** (per chain, in `build-data.json`/SystemControl) before minting — if `true`, mints revert.
- **Protocol state (beta).** Beyond `mintingStopped`, the protocol can enter an **Emergency** state that suspends new deposits while still allowing withdrawals. Treat a mint revert as possibly state-driven, and keep burn/withdraw paths available even when minting is off.
- **SIR is per-chain.** Each chain issues its own SIR (~2.015B/yr) with no cross-chain bridge — balances, staking, and rewards never move between chains. A portfolio tracker must read each chain separately.
- **No UI guardrails.** Direct contract calls bypass the app's vault blocklist (pairs the UI disables for thin liquidity). Before minting, sanity-check the vault's reserves and saturation via `getReserves` / subgraph; a saturated or illiquid vault is a bad entry.
- **Always quote and set slippage/min-out** for debt-token deposits (`collateralToDepositMin`) — these route through the DEX.
- **Set a real `deadline`** (unix seconds) on `mint`/`burn`; `0` disables the check.
- **Amount parsing:** use `parseUnits`/`formatUnits` with the correct decimals (SIR=12, APE=`ape.decimals`, TEA=collateral decimals). Never round-trip token amounts through JS `Number`.

## Resources

- App: https://app.sir.trading · Live data: https://app.sir.trading/build-data.json
- Docs: https://docs.sir.trading
- Core contracts (Vault, SIR/Staker, TEA, APE, Oracle): https://github.com/SIR-trading/Core
- Periphery (Assistant, Quoter, Treasury): https://github.com/SIR-trading/Periphery
- This skill: https://github.com/SIR-trading/sir-trading-skill
