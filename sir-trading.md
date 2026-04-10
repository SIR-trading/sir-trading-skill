# Sir Trading on MegaETH

AI coding skill for integrating [Sir Trading](https://app.sir.trading), a no-liquidation leveraged token protocol on MegaETH. Covers going long (minting APE), providing liquidity (minting TEA), closing/reducing positions (burning), quoting prices, creating new pairs (vaults), MegaSIR staking, claiming rewards, fee auctions, pair discovery, trading bots, and portfolio tracking.

## Why SIR

SIR is a new primitive for holding leverage long-term. Margin has liquidation and ongoing borrow fees. Leveraged tokens suffer volatility decay from rebalancing. Perps have liquidation and funding fees. Options have time decay and expiry. SIR eliminates all of these: no liquidation, no funding fees while holding, and best-effort mitigation of volatility decay. Fees are charged only on mint and burn. APE holders get a convex payoff (compounding gains up, bounded downside down). LPs (TEA holders) take the other side and collect trading fees, similar to Uniswap LPing.

## What This Skill Is For

Use this skill when the user asks for:
- Going long / opening a leveraged position (minting APE)
- Closing or reducing a position (burning APE)
- Providing liquidity / LPing (minting TEA)
- Withdrawing liquidity (burning TEA)
- Quoting expected output before minting/burning
- Creating new pairs (initializing vaults)
- Discovering existing pairs (vaults) and reading reserves
- Staking or unstaking MegaSIR
- Claiming staking dividends or LP rewards
- Starting or bidding on fee auctions
- Building trading bots or portfolio trackers
- Reading user positions (Ledger) and unclaimed rewards

## Architecture

Sir Trading is a no-liquidation leveraged token protocol. The app UI uses friendly terms; see the **Terminology** table below for the full mapping between app labels and contract terms. Key concepts:

- **Vaults (called "Pairs" in the app UI)** are identified by `VaultParameters { debtToken, collateralToken, leverageTier }`. A singleton Vault contract holds all vaults.
- **APE** = ERC-20 leveraged token (going long). Each vault deploys a separate APE clone contract with dynamic per-vault naming (not a single "MegaAPE"). Get the address via `Assistant.getAddressAPE(vaultId)`.
- **TEA** = ERC-1155 LP token (providing liquidity). The Vault contract itself is the TEA token. Use the `vaultId` as the ERC-1155 `tokenId`.
- **No liquidation:** APE has a convex payoff with bounded downside. APE holders cannot be liquidated.
- **Leverage formula:** `leverage = 1 + 2^tier`. Tiers range from -4 (1.0625x) to +2 (5x).
- **MegaSIR has 12 decimals, not 18.** Use `parseUnits(amount, 12)`, not `parseEther`.
- **Oracle** uses a 30-minute TWAP from Kumbaya DEX (Uniswap V3 fork) pools.

## Terminology

The app UI at [app.sir.trading](https://app.sir.trading) uses friendly labels. Both the app terms and contract terms appear throughout this skill.

| App UI Label | Contract Term | Description |
|---|---|---|
| **Pair** | Vault | A leveraged token tracking collateral vs debt (base vs quote) at a specific constant leverage |
| **Go Long** | Mint APE | Open a leveraged position |
| **Close Position** / **Reduce Position** | Burn APE | Fully or partially exit a long |
| **Provide Liquidity** | Mint TEA | Deposit collateral as an LP |
| **Withdraw** / **Withdraw All** | Burn TEA | Exit an LP position |
| **New Pair** | Initialize vault | Create a new vault |
| **Longs** | Active APE positions | Tab in Ledger & Leaderboard |
| **LP** | Active TEA positions | Tab in Ledger & Leaderboard |
| **Stake** / **Unstake** | stake / unstake | MegaSIR token staking (same in UI and contract) |
| **Claim MegaSIR Rewards** | lperMint / claim | Claim MegaSIR or ETH dividends |
| **Ledger** | User positions page | Open longs, LP, staking, closed positions |
| **The Exchange** | Leverage page | Page header for going long |
| **The Tea Room** | Liquidity page | Page header for providing liquidity |
| **Auctions** | Fee auctions page | Bidding on accumulated fees |

## Chain Configuration

| Network | Chain ID | RPC |
|---------|----------|-----|
| MegaETH Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` |

## Contract Addresses

### Core Protocol (MegaETH 4326)

| Contract | Address |
|----------|---------|
| Vault | `0x8d694D1b369BdE5B274Ad643fEdD74f836E88543` |
| MegaSIR Token | `0x9367A0c482703d8d9bda995B03f8E71056a72500` |
| Oracle | `0x4edF071a7dEe52fBE663DF7873994725ba91Cdc7` |
| SystemControl | `0x549618c8E4b74f9eB519e459698b2CaF53dA0453` |
| Contributors | `0x686748764c5C7Aa06FEc784E60D14b650bF79129` |
| Assistant | `0xB91AE2c8365FD45030abA84a4666C4dB074E53E7` |
| Treasury | `0xf1Db8f4D2543c2C881a3dC90754CEDf549cD362d` |

### Kumbaya DEX Dependencies

| Contract | Address |
|----------|---------|
| UniswapV3Factory | `0x68b34591f662508076927803c567Cc8006988a09` |
| SwapRouter | `0xE5BbEF8De2DB447a7432A47EBa58924d94eE470e` |
| QuoterV2 | `0x1F1a8dC7E138C34b503Ca080962aC10B75384a27` |
| WETH | `0x4200000000000000000000000000000000000006` |

## ABIs

### Vault ABI

```typescript
export const vaultAbi = [
  // Mint APE or TEA
  {
    type: "function",
    name: "mint",
    inputs: [
      { name: "isAPE", type: "bool" },
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
      { name: "amountToDeposit", type: "uint256" },
      { name: "collateralToDepositMin", type: "uint144" },
      { name: "deadline", type: "uint40" },
      { name: "portionLockTime", type: "uint8" },
    ],
    outputs: [{ name: "amount", type: "uint256" }],
    stateMutability: "payable",
  },
  // Burn APE or TEA
  {
    type: "function",
    name: "burn",
    inputs: [
      { name: "isAPE", type: "bool" },
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
      { name: "amount", type: "uint256" },
      { name: "deadline", type: "uint40" },
    ],
    outputs: [{ name: "", type: "uint144" }],
    stateMutability: "nonpayable",
  },
  // Initialize a new vault
  {
    type: "function",
    name: "initialize",
    inputs: [
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
    ],
    outputs: [],
    stateMutability: "nonpayable",
  },
  // Get reserves for a vault
  {
    type: "function",
    name: "getReserves",
    inputs: [
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
    ],
    outputs: [
      {
        name: "",
        type: "tuple",
        components: [
          { name: "reserveApes", type: "uint144" },
          { name: "reserveLPers", type: "uint144" },
          { name: "tickPriceX42", type: "int64" },
        ],
      },
    ],
    stateMutability: "view",
  },
  // Get vault state
  {
    type: "function",
    name: "vaultStates",
    inputs: [
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
    ],
    outputs: [
      {
        name: "",
        type: "tuple",
        components: [
          { name: "reserve", type: "uint144" },
          { name: "tickPriceSatX42", type: "int64" },
          { name: "vaultId", type: "uint48" },
        ],
      },
    ],
    stateMutability: "view",
  },
  // Get vault parameters by ID
  {
    type: "function",
    name: "paramsById",
    inputs: [{ name: "vaultId", type: "uint48" }],
    outputs: [
      {
        name: "",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
    ],
    stateMutability: "view",
  },
  // Total number of vaults
  {
    type: "function",
    name: "numberOfVaults",
    inputs: [],
    outputs: [{ name: "", type: "uint48" }],
    stateMutability: "view",
  },
  // Total reserves for a collateral token
  {
    type: "function",
    name: "totalReserves",
    inputs: [{ name: "collateral", type: "address" }],
    outputs: [{ name: "", type: "uint256" }],
    stateMutability: "view",
  },
  // TEA balance (ERC-1155)
  {
    type: "function",
    name: "balanceOf",
    inputs: [
      { name: "account", type: "address" },
      { name: "vaultId", type: "uint256" },
    ],
    outputs: [{ name: "", type: "uint256" }],
    stateMutability: "view",
  },
  // TEA transfer (ERC-1155)
  {
    type: "function",
    name: "safeTransferFrom",
    inputs: [
      { name: "from", type: "address" },
      { name: "to", type: "address" },
      { name: "vaultId", type: "uint256" },
      { name: "amount", type: "uint256" },
      { name: "data", type: "bytes" },
    ],
    outputs: [],
    stateMutability: "nonpayable",
  },
  // TEA approval (ERC-1155)
  {
    type: "function",
    name: "setApprovalForAll",
    inputs: [
      { name: "operator", type: "address" },
      { name: "approved", type: "bool" },
    ],
    outputs: [],
    stateMutability: "nonpayable",
  },
  // TEA total supply per vault
  {
    type: "function",
    name: "totalSupply",
    inputs: [{ name: "vaultId", type: "uint256" }],
    outputs: [{ name: "", type: "uint256" }],
    stateMutability: "view",
  },
  // Unclaimed MegaSIR rewards for an LP
  {
    type: "function",
    name: "unclaimedRewards",
    inputs: [
      { name: "vaultId", type: "uint256" },
      { name: "lper", type: "address" },
    ],
    outputs: [{ name: "", type: "uint80" }],
    stateMutability: "view",
  },
  // TEA lock end timestamp
  {
    type: "function",
    name: "lockEnd",
    inputs: [
      { name: "account", type: "address" },
      { name: "vaultId", type: "uint256" },
    ],
    outputs: [{ name: "", type: "uint40" }],
    stateMutability: "view",
  },
] as const;
```

### MegaSIR / Staker ABI

> **Note:** `bid()` is `payable` on MegaETH; send ETH as `msg.value`.

```typescript
export const sirAbi = [
  // Staking
  {
    type: "function",
    name: "stake",
    inputs: [{ name: "amount", type: "uint80" }],
    outputs: [],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "unstake",
    inputs: [{ name: "amount", type: "uint80" }],
    outputs: [],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "claim",
    inputs: [],
    outputs: [{ name: "dividends_", type: "uint96" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "unstakeAndClaim",
    inputs: [{ name: "amount", type: "uint80" }],
    outputs: [{ name: "dividends_", type: "uint96" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "stakeOf",
    inputs: [{ name: "staker", type: "address" }],
    outputs: [
      { name: "unlockedStake", type: "uint80" },
      { name: "lockedStake", type: "uint80" },
    ],
    stateMutability: "view",
  },
  {
    type: "function",
    name: "unclaimedDividends",
    inputs: [{ name: "staker", type: "address" }],
    outputs: [{ name: "", type: "uint96" }],
    stateMutability: "view",
  },
  // LP reward claiming
  {
    type: "function",
    name: "lperMint",
    inputs: [{ name: "vaultId", type: "uint256" }],
    outputs: [{ name: "rewards", type: "uint80" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "lperMintAndStake",
    inputs: [{ name: "vaultId", type: "uint256" }],
    outputs: [{ name: "rewards", type: "uint80" }],
    stateMutability: "nonpayable",
  },
  // Contributor reward claiming
  {
    type: "function",
    name: "contributorMint",
    inputs: [],
    outputs: [{ name: "rewards", type: "uint80" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "contributorMintAndStake",
    inputs: [],
    outputs: [{ name: "rewards", type: "uint80" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "contributorUnclaimedSIR",
    inputs: [{ name: "contributor", type: "address" }],
    outputs: [{ name: "", type: "uint80" }],
    stateMutability: "view",
  },
  // Fee auctions
  {
    type: "function",
    name: "collectFeesAndStartAuction",
    inputs: [{ name: "token", type: "address" }],
    outputs: [{ name: "", type: "uint256" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "bid",
    inputs: [
      { name: "token", type: "address" },
      { name: "amount", type: "uint96" },
    ],
    outputs: [],
    stateMutability: "payable",
  },
  {
    type: "function",
    name: "getAuctionLot",
    inputs: [
      { name: "token", type: "address" },
      { name: "beneficiary", type: "address" },
    ],
    outputs: [],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "auctions",
    inputs: [{ name: "token", type: "address" }],
    outputs: [
      { name: "bidder", type: "address" },
      { name: "bid", type: "uint96" },
      { name: "startTime", type: "uint40" },
    ],
    stateMutability: "view",
  },
  // ERC-20
  {
    type: "function",
    name: "balanceOf",
    inputs: [{ name: "account", type: "address" }],
    outputs: [{ name: "", type: "uint256" }],
    stateMutability: "view",
  },
  {
    type: "function",
    name: "approve",
    inputs: [
      { name: "spender", type: "address" },
      { name: "amount", type: "uint256" },
    ],
    outputs: [{ name: "", type: "bool" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "transfer",
    inputs: [
      { name: "to", type: "address" },
      { name: "amount", type: "uint256" },
    ],
    outputs: [{ name: "", type: "bool" }],
    stateMutability: "nonpayable",
  },
  {
    type: "function",
    name: "decimals",
    inputs: [],
    outputs: [{ name: "", type: "uint8" }],
    stateMutability: "view",
  },
  // Multicall
  {
    type: "function",
    name: "multicall",
    inputs: [{ name: "data", type: "bytes[]" }],
    outputs: [{ name: "results", type: "bytes[]" }],
    stateMutability: "nonpayable",
  },
] as const;
```

### Assistant ABI

```typescript
export const assistantAbi = [
  // Quote minting with collateral token
  {
    type: "function",
    name: "quoteMint",
    inputs: [
      { name: "isAPE", type: "bool" },
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
      { name: "amountCollateral", type: "uint144" },
    ],
    outputs: [{ name: "amountTokens", type: "uint256" }],
    stateMutability: "view",
  },
  // Quote minting with debt token (includes Kumbaya swap quote)
  {
    type: "function",
    name: "quoteMintWithDebtToken",
    inputs: [
      { name: "isAPE", type: "bool" },
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
      { name: "amountDebtToken", type: "uint256" },
    ],
    outputs: [
      { name: "amountTokens", type: "uint256" },
      { name: "amountCollateral", type: "uint256" },
      { name: "amountCollateralIdeal", type: "uint256" },
    ],
    stateMutability: "view",
  },
  // Quote burning APE or TEA
  {
    type: "function",
    name: "quoteBurn",
    inputs: [
      { name: "isAPE", type: "bool" },
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
      { name: "amountTokens", type: "uint256" },
    ],
    outputs: [
      { name: "amountCollateral", type: "uint144" },
      { name: "amountDebtToken", type: "uint256" },
    ],
    stateMutability: "view",
  },
  // Convert collateral amount to debt token equivalent
  {
    type: "function",
    name: "quoteCollateralToDebtToken",
    inputs: [
      { name: "debtToken", type: "address" },
      { name: "collateralToken", type: "address" },
      { name: "amountCollateral", type: "uint256" },
    ],
    outputs: [{ name: "amountDebtToken", type: "uint256" }],
    stateMutability: "view",
  },
  // Get reserves for multiple vaults by ID
  {
    type: "function",
    name: "getReserves",
    inputs: [{ name: "vaultIds", type: "uint48[]" }],
    outputs: [
      {
        name: "reserves",
        type: "tuple[]",
        components: [
          { name: "reserveApes", type: "uint144" },
          { name: "reserveLPers", type: "uint144" },
          { name: "tickPriceX42", type: "int64" },
        ],
      },
    ],
    stateMutability: "view",
  },
  // Get user balances across vaults (APE, TEA, unclaimed MegaSIR)
  {
    type: "function",
    name: "getUserBalances",
    inputs: [
      { name: "user", type: "address" },
      { name: "offset", type: "uint256" },
      { name: "numVaults", type: "uint256" },
    ],
    outputs: [
      { name: "apeBalances", type: "uint256[]" },
      { name: "teaBalances", type: "uint256[]" },
      { name: "unclaimedSirRewards", type: "uint80[]" },
    ],
    stateMutability: "view",
  },
  // Price of TEA in collateral (num/den)
  {
    type: "function",
    name: "priceOfTEA",
    inputs: [
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
    ],
    outputs: [
      { name: "num", type: "uint256" },
      { name: "den", type: "uint256" },
    ],
    stateMutability: "view",
  },
  // Price of APE in collateral (num/den)
  {
    type: "function",
    name: "priceOfAPE",
    inputs: [
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
    ],
    outputs: [
      { name: "num", type: "uint256" },
      { name: "den", type: "uint256" },
    ],
    stateMutability: "view",
  },
  // Check vault status (InvalidVault, NoUniswapPool, VaultCanBeCreated, VaultAlreadyExists)
  {
    type: "function",
    name: "getVaultStatus",
    inputs: [
      {
        name: "vaultParams",
        type: "tuple",
        components: [
          { name: "debtToken", type: "address" },
          { name: "collateralToken", type: "address" },
          { name: "leverageTier", type: "int8" },
        ],
      },
    ],
    outputs: [{ name: "", type: "uint8" }],
    stateMutability: "view",
  },
  // Get the APE token address for a vault
  {
    type: "function",
    name: "getAddressAPE",
    inputs: [{ name: "vaultId", type: "uint48" }],
    outputs: [{ name: "", type: "address" }],
    stateMutability: "view",
  },
] as const;
```

## Default Stack Decisions (Opinionated)

### 1. Assistant for quotes, Vault for execution
The Assistant contract is read-only. Use it for `quoteMint`, `quoteBurn`, `priceOfAPE`, `getUserBalances`, etc. The Vault contract executes state-changing operations (`mint`, `burn`, `initialize`).

### 2. MegaSIR is 12 decimals
Use `parseUnits(amount, 12)`, not `parseEther`. The MegaSIR token uses 12 decimal places.

### 3. VaultParameters is a struct tuple
Always pass as `{ debtToken: address, collateralToken: address, leverageTier: int8 }`. This struct identifies a unique vault.

### 4. APE addresses are per-vault
Get the APE ERC-20 address via `Assistant.getAddressAPE(vaultId)`. TEA uses the Vault contract address with the `vaultId` as the ERC-1155 `tokenId`.

## Core Operations

### Client Setup

```typescript
import { createPublicClient, createWalletClient, http, defineChain, parseEther, parseUnits } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { vaultAbi, sirAbi, assistantAbi } from './abis'

const megaeth = defineChain({
  id: 4326,
  name: 'MegaETH',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: ['https://mainnet.megaeth.com/rpc'] } },
})

const VAULT = '0x8d694D1b369BdE5B274Ad643fEdD74f836E88543'
const SIR = '0x9367A0c482703d8d9bda995B03f8E71056a72500'
const ASSISTANT = '0xB91AE2c8365FD45030abA84a4666C4dB074E53E7'
const WETH = '0x4200000000000000000000000000000000000006'

const publicClient = createPublicClient({ chain: megaeth, transport: http() })
const account = privateKeyToAccount('0x...')
const walletClient = createWalletClient({ account, chain: megaeth, transport: http() })
```

### Discover Vaults (Pairs)

```typescript
const numVaults = await publicClient.readContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'numberOfVaults',
})

// Iterate all vaults
for (let i = 1; i <= numVaults; i++) {
  const params = await publicClient.readContract({
    address: VAULT,
    abi: vaultAbi,
    functionName: 'paramsById',
    args: [i],
  })
  console.log(`Vault ${i}:`, params.debtToken, params.collateralToken, params.leverageTier)
}
```

### Check Vault Status

```typescript
// VaultStatus enum: 0=InvalidVault, 1=NoUniswapPool, 2=VaultCanBeCreated, 3=VaultAlreadyExists
const status = await publicClient.readContract({
  address: ASSISTANT,
  abi: assistantAbi,
  functionName: 'getVaultStatus',
  args: [{ debtToken: WETH, collateralToken: SOME_TOKEN, leverageTier: 1 }],
})
```

### Create a New Vault (New Pair)

A vault can be created if `getVaultStatus` returns `2` (VaultCanBeCreated). This requires a Kumbaya DEX pool to exist for the collateral/debt pair.

```typescript
const vaultParams = { debtToken: WETH, collateralToken: SOME_TOKEN, leverageTier: 1 }

// 1. Check status
const status = await publicClient.readContract({
  address: ASSISTANT,
  abi: assistantAbi,
  functionName: 'getVaultStatus',
  args: [vaultParams],
})
if (status !== 2) throw new Error('Vault cannot be created (status: ' + status + ')')

// 2. Initialize
await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'initialize',
  args: [vaultParams],
})
```

### Quote Mint APE

```typescript
const vaultParams = { debtToken: WETH, collateralToken: SOME_TOKEN, leverageTier: 1 }
const collateralAmount = parseUnits('100', 18) // adjust decimals to collateral token

const expectedAPE = await publicClient.readContract({
  address: ASSISTANT,
  abi: assistantAbi,
  functionName: 'quoteMint',
  args: [true, vaultParams, collateralAmount],
})
```

### Quote Mint with Debt Token

Quotes the swap from debt token to collateral via Kumbaya DEX, then quotes the resulting APE/TEA amount.

```typescript
const [amountTokens, amountCollateral, amountCollateralIdeal] = await publicClient.readContract({
  address: ASSISTANT,
  abi: assistantAbi,
  functionName: 'quoteMintWithDebtToken',
  args: [true, vaultParams, parseEther('1')], // 1 WETH as debt token
})
// amountTokens = expected APE/TEA output
// amountCollateral = collateral after swap (with slippage)
// amountCollateralIdeal = collateral at spot price (no slippage)
```

### Quote Burn

```typescript
const [amountCollateral, amountDebtToken] = await publicClient.readContract({
  address: ASSISTANT,
  abi: assistantAbi,
  functionName: 'quoteBurn',
  args: [true, vaultParams, apeAmount], // isAPE=true for APE, false for TEA
})
// amountCollateral = collateral you'd receive
// amountDebtToken = equivalent in debt token (TWAP price)
```

### Mint APE (Go Long) with Collateral Token

The standard flow. Deposit collateral directly; no swap needed. Set `collateralToDepositMin` to `0n` since you're depositing collateral directly.

```typescript
const erc20Abi = [
  {
    type: 'function', name: 'approve', stateMutability: 'nonpayable',
    inputs: [{ name: 'spender', type: 'address' }, { name: 'amount', type: 'uint256' }],
    outputs: [{ name: '', type: 'bool' }],
  },
] as const

const collateralToken = '0x...' // the vault's collateral token
const collateralAmount = parseUnits('100', 18)
const deadline = BigInt(Math.floor(Date.now() / 1000) + 600) // 10 min

// 1. Approve collateral
await walletClient.writeContract({
  address: collateralToken,
  abi: erc20Abi,
  functionName: 'approve',
  args: [VAULT, collateralAmount],
})

// 2. Mint APE
await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'mint',
  args: [true, vaultParams, collateralAmount, 0n, deadline, 0],
})
```

### Mint APE (Go Long) with Debt Token

The vault automatically swaps the debt token to collateral via Kumbaya DEX. When `collateralToDepositMin > 0`, the contract knows `amountToDeposit` is in debt token and performs the swap. Use `quoteMintWithDebtToken` to get the expected collateral output for setting the minimum.

```typescript
// 1. Quote to get expected collateral after swap
const [, quotedCollateral] = await publicClient.readContract({
  address: ASSISTANT,
  abi: assistantAbi,
  functionName: 'quoteMintWithDebtToken',
  args: [true, vaultParams, debtAmount],
})
const minCollateral = quotedCollateral * 99n / 100n // 1% slippage

// 2. Approve debt token
await walletClient.writeContract({
  address: debtToken,
  abi: erc20Abi,
  functionName: 'approve',
  args: [VAULT, debtAmount],
})

// 3. Mint APE (collateralToDepositMin > 0 signals debt-token deposit)
await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'mint',
  args: [true, vaultParams, debtAmount, minCollateral, deadline, 0],
})
```

### Mint APE (Go Long) with Native ETH

If the vault uses WETH as collateral or debt token, send ETH as `msg.value` and the contract wraps it automatically.

```typescript
// WETH-collateral vault: deposit ETH directly as collateral
await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'mint',
  args: [true, vaultParams, 0n, 0n, deadline, 0],
  value: parseEther('1'),
})
```

### Mint TEA (Provide Liquidity) with Collateral

Same as APE but with `isAPE=false`. The optional `portionLockTime` (0-255) reduces the TEA minting fee: 0 means no lock (full fee), 255 means maximum lock (zero fee). The max lock duration defaults to 90 days.

```typescript
// Approve collateral first (same as APE flow above)

await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'mint',
  args: [false, vaultParams, collateralAmount, 0n, deadline, 128], // ~50% lock for ~50% fee reduction
})
```

### Mint TEA (Provide Liquidity) with Debt Token

Same debt-token swap mechanism as APE minting. `collateralToDepositMin > 0` triggers the swap.

```typescript
await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'mint',
  args: [false, vaultParams, debtAmount, minCollateral, deadline, 0],
})
```

### Mint TEA (Provide Liquidity) with Native ETH

```typescript
await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'mint',
  args: [false, vaultParams, 0n, 0n, deadline, 0],
  value: parseEther('1'),
})
```

### Burn APE (Close/Reduce Position)

Returns collateral to the caller.

```typescript
await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'burn',
  args: [true, vaultParams, apeAmount, deadline],
})
```

### Burn TEA (Withdraw Liquidity)

Returns collateral. TEA must not be locked (check `lockEnd` first).

```typescript
// Check if TEA is unlocked
const lockEndTime = await publicClient.readContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'lockEnd',
  args: [account.address, vaultId],
})
if (lockEndTime > BigInt(Math.floor(Date.now() / 1000))) {
  throw new Error('TEA is still locked until ' + new Date(Number(lockEndTime) * 1000))
}

await walletClient.writeContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'burn',
  args: [false, vaultParams, teaAmount, deadline],
})
```

### Read Vault Reserves

```typescript
const reserves = await publicClient.readContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'getReserves',
  args: [vaultParams],
})
// reserves.reserveApes — collateral owned by APE holders
// reserves.reserveLPers — collateral owned by TEA (LP) holders
// reserves.tickPriceX42 — log_{1.0001}(price) scaled by 2^42
```

### Read User Positions

Batch-read APE balances, TEA balances, and unclaimed MegaSIR rewards across all vaults.

```typescript
const numVaults = await publicClient.readContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'numberOfVaults',
})

// getUserBalances reads vaults [offset+1, offset+numVaults]
const [apeBalances, teaBalances, unclaimedSirRewards] = await publicClient.readContract({
  address: ASSISTANT,
  abi: assistantAbi,
  functionName: 'getUserBalances',
  args: [userAddress, 0n, BigInt(numVaults)],
})

// Check individual vault LP rewards
const pendingRewards = await publicClient.readContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'unclaimedRewards',
  args: [vaultId, userAddress],
})

// Check TEA lock status
const lockEndTimestamp = await publicClient.readContract({
  address: VAULT,
  abi: vaultAbi,
  functionName: 'lockEnd',
  args: [userAddress, vaultId],
})
const isLocked = lockEndTimestamp > BigInt(Math.floor(Date.now() / 1000))
```

### Stake MegaSIR

```typescript
const amount = parseUnits('100', 12) // 12 decimals!

// Stake (uses internal balance accounting, no approve needed)
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'stake',
  args: [amount],
})
```

### Check Stake Status

Locked stake unlocks progressively with a 30-day half-life: after 30 days half is unlocked, after 60 days 75%, etc. Only the unlocked portion can be unstaked.

```typescript
const [unlockedStake, lockedStake] = await publicClient.readContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'stakeOf',
  args: [account.address],
})
// unlockedStake — can be unstaked now
// lockedStake — still locked, unlocking with 30-day half-life
```

### Unstake MegaSIR

Can only unstake up to the unlocked portion. Reverts if amount exceeds unlocked stake.

```typescript
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'unstake',
  args: [amount], // must be <= unlockedStake
})
```

### Claim Staking Dividends

Claims accrued ETH dividends without unstaking.

```typescript
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'claim',
})
```

### Unstake and Claim in One Transaction

```typescript
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'unstakeAndClaim',
  args: [amount],
})
```

### Claim LP Rewards

```typescript
// Claim MegaSIR rewards for providing TEA liquidity in a vault
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'lperMint',
  args: [vaultId],
})

// Or claim and stake in one tx
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'lperMintAndStake',
  args: [vaultId],
})
```

### Start a Fee Auction

Anyone can call this to initiate an auction for accumulated fees in a given token. Auctions last 24 hours, with a 247-hour cooldown between auctions for the same token.

```typescript
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'collectFeesAndStartAuction',
  args: [tokenAddress],
})
```

### Bid on an Auction

Must exceed the previous bid by at least 5% on MegaETH. Send ETH as `msg.value`.

```typescript
// Check current auction state
const [bidder, currentBid, startTime] = await publicClient.readContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'auctions',
  args: [tokenAddress],
})

// Place a bid (must be >= previous bid * 1.05)
const bidAmount = currentBid * 105n / 100n + 1n // 5% above previous
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'bid',
  args: [tokenAddress, bidAmount],
  value: bidAmount,
})
```

### Claim Auction Lot

Winner calls after the 24-hour auction ends to receive the auctioned tokens.

```typescript
await walletClient.writeContract({
  address: SIR,
  abi: sirAbi,
  functionName: 'getAuctionLot',
  args: [tokenAddress, beneficiaryAddress],
})
```

## Foundry Integration (Solidity)

### Mint APE (Go Long) from Collateral

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IERC20} from "forge-std/interfaces/IERC20.sol";

interface IVault {
    struct VaultParameters {
        address debtToken;
        address collateralToken;
        int8 leverageTier;
    }

    function mint(
        bool isAPE,
        VaultParameters calldata vaultParams,
        uint256 amountToDeposit,
        uint144 collateralToDepositMin,
        uint40 deadline,
        uint8 portionLockTime
    ) external payable returns (uint256);
}

contract SirMinter {
    IVault constant VAULT = IVault(0x8d694D1b369BdE5B274Ad643fEdD74f836E88543);

    function mintAPE(
        IVault.VaultParameters calldata vaultParams,
        uint256 collateralAmount
    ) external returns (uint256 apeAmount) {
        IERC20(vaultParams.collateralToken).transferFrom(msg.sender, address(this), collateralAmount);
        IERC20(vaultParams.collateralToken).approve(address(VAULT), collateralAmount);

        apeAmount = VAULT.mint(
            true,                           // isAPE
            vaultParams,
            collateralAmount,
            0,                              // collateralToDepositMin (0 = collateral deposit)
            uint40(block.timestamp + 600),  // deadline
            0                               // portionLockTime (ignored for APE)
        );
    }
}
```

### Read Reserves and Quote Burn via Assistant

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IAssistant {
    struct VaultParameters {
        address debtToken;
        address collateralToken;
        int8 leverageTier;
    }

    struct Reserves {
        uint144 reserveApes;
        uint144 reserveLPers;
        int64 tickPriceX42;
    }

    function getReserves(uint48[] calldata vaultIds) external view returns (Reserves[] memory);
    function quoteBurn(bool isAPE, VaultParameters calldata vaultParams, uint256 amountTokens)
        external view returns (uint144 amountCollateral, uint256 amountDebtToken);
}

contract SirReader {
    IAssistant constant ASSISTANT = IAssistant(0xB91AE2c8365FD45030abA84a4666C4dB074E53E7);

    function getReservesAndQuoteBurn(
        uint48 vaultId,
        IAssistant.VaultParameters calldata vaultParams,
        uint256 apeAmount
    ) external view returns (
        IAssistant.Reserves memory reserves,
        uint144 collateralOut,
        uint256 debtTokenOut
    ) {
        uint48[] memory ids = new uint48[](1);
        ids[0] = vaultId;
        reserves = ASSISTANT.getReserves(ids)[0];
        (collateralOut, debtTokenOut) = ASSISTANT.quoteBurn(true, vaultParams, apeAmount);
    }
}
```

## Fee Structure Reference

| Fee | Rate | Recipient |
|-----|------|-----------|
| APE mint/burn | Same fee applies independently on both mint and burn: `feeRate = (baseFee × 2^tier) / (10000 + baseFee × 2^tier)` with default `baseFee = 1000`. Per-operation examples: tier -2 (1.0625x) ~2.4%, tier 0 (2x) ~9.1%, tier 1 (3x) ~16.7%, tier 2 (5x) ~28.6%. `priceOfAPE` and `quoteBurn` already account for the burn fee. | LPers + Stakers (stakers get up to 50% depending on vault tax) |
| TEA mint | Default ~4.9% (`lpFee` 515 bps). **Reducible to 0% with full lock** (max lock defaults to 90 days) | Protocol-Owned Liquidity |
| TEA burn | 0% | — |
| Staker dividends | Distributed via fee auctions (24h duration, 247h cooldown) | MegaSIR stakers (in ETH) |

## MegaETH-Specific Notes

- **Instant receipts:** Use `eth_sendRawTransactionSync` for sub-10ms confirmation via EIP-7966.
- **No public mempool:** Transactions are sequencer-ordered. Front-running risk is structurally lower.
- **Gas model:** SSTORE costs differ from L1. First vault interactions are more expensive (new storage slots).
- **MegaSIR has 12 decimals**, not 18. Always use `parseUnits(amount, 12)`.
- **APE tokens have dynamic per-vault naming:** each vault deploys a separate APE clone. There is no single "MegaAPE" token.
- **Optional TEA lock for fee reduction:** `portionLockTime` 0-255, where 0 = no lock (full fee), 255 = max lock (zero fee).
- **Oracle:** 30-minute TWAP from Kumbaya DEX pools.
- **Auction minimum bid increase:** 5%.
- **Manipulation resistance:** Fees on both mint and burn plus a long 30-minute TWAP minimize price manipulation attacks.
- **Saturation risk:** When creating vaults, if leverage is too high for the pair's volatility, the vault will frequently enter saturation, making it unappealing for both traders and LPs.

## Resources

- [Sir Trading App](https://app.sir.trading): Web interface
- [Sir Trading Docs](https://docs.sir.trading): Protocol documentation
- [Core Contracts](https://github.com/SIR-trading/Core): Vault, SIR/Staker, TEA, APE, Oracle
- [Periphery Contracts](https://github.com/SIR-trading/Periphery): Assistant, Quoter, Treasury
- Kumbaya DEX skill: for swap integration and pool discovery
