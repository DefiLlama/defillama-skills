---
name: stablecoin-dynamics
description: >
  Stablecoin market structure analysis using DefiLlama MCP tools. Covers
  chain-level dominance computation, supply flow tracking across periods,
  concentration risk assessment, peg mechanism comparison, and depeg
  monitoring. Use when users ask about stablecoin market share, which
  chains are gaining or losing stablecoin supply, stablecoin dominance,
  USDT vs USDC distribution, depeg risk, or stablecoin flow trends.
---

# Stablecoin Dynamics

Analyze stablecoin market structure, supply distribution, and flow
patterns by composing multiple tool calls into derived metrics the
raw data doesn't surface directly.

## Chain Dominance

Compute what percentage of a chain's stablecoin supply each stablecoin
holds. This requires two calls and a calculation.

### Step 1 -- Get all stablecoins on the target chain

```
defillama:get_stablecoin_supply
  chain: "<chain>"
  sort_by: "circulating_supply desc"
```

### Step 2 -- Compute dominance

Sum `circulating_supply` across all rows for the chain total. Each
stablecoin's dominance = `circulating_supply / chain_total * 100`.

Present as a table:

| Stablecoin | Supply | Dominance |
|------------|--------|-----------|
| USDT       | $82.4B | 50.9%     |
| USDC       | $49.5B | 30.6%     |
| DAI        | $12.1B | 7.5%      |
| ...        | ...    | ...       |

### Concentration flag

If any single stablecoin holds > 60% dominance on a chain, flag it as
a concentration risk. High single-issuer dependence means a depeg or
regulatory action on that issuer disproportionately impacts the chain's
DeFi ecosystem.

## Supply Flow Analysis

Track which chains are gaining or losing stablecoin market cap over a
period. Requires historical queries and delta computation.

### Step 1 -- Current supply by chain

```
defillama:get_stablecoin_supply
  sort_by: "circulating_supply desc"
  limit: 30
```

This returns current supply per stablecoin per chain. Aggregate by
chain (sum all stablecoins per chain) to get each chain's total
stablecoin supply.

### Step 2 -- Historical supply

```
defillama:get_stablecoin_supply
  period: "30d"
  sort_by: "circulating_supply desc"
```

With `period` set, this returns daily snapshots. Take the earliest date
as the period start and the latest as the period end.

### Step 3 -- Compute deltas

For each chain: `change = current_supply - supply_at_period_start`.
`change_pct = change / supply_at_period_start * 100`.

Present sorted by absolute change or percentage change:

| Chain     | Current     | 30d Ago     | Change      | Change % |
|-----------|-------------|-------------|-------------|----------|
| Solana    | $12.8B      | $10.1B      | +$2.7B      | +26.7%   |
| Base      | $8.2B       | $6.9B       | +$1.3B      | +18.8%   |
| Ethereum  | $82.4B      | $83.1B      | -$0.7B      | -0.8%    |
| Tron      | $61.2B      | $62.8B      | -$1.6B      | -2.5%    |

### Interpretation

- Chains gaining stablecoin supply are attracting capital (bullish for
  on-chain activity and DeFi TVL growth).
- Chains losing supply may signal capital rotation, declining DeFi
  yields, or regulatory pressure.
- Compare with `defillama:get_chain_metrics` TVL trends to see if
  stablecoin flows lead or lag TVL changes.

## Peg Mechanism Comparison

Compare stablecoins by their peg mechanism to assess systemic risk
exposure across a chain or the market.

### Step 1 -- Get stablecoins with peg type

```
defillama:get_stablecoin_supply
  sort_by: "circulating_supply desc"
  limit: 50
```

The `peg_type` field classifies each stablecoin (usd, eur, etc.).
Group by the issuer/mechanism type to assess exposure:

- **Fiat-backed centralized** (USDT, USDC, BUSD): counterparty risk
  on issuer, regulatory exposure
- **Crypto-collateralized** (DAI, LUSD, crvUSD): smart contract risk,
  liquidation cascade risk
- **Algorithmic** (FRAX, UST-style): depeg risk under redemption pressure

### Step 2 -- Cross-reference with chain data

```
defillama:get_chain_metrics
  chain: "<chain>"
```

Compare a chain's stablecoin composition with its DeFi TVL. A chain
where 90% of stablecoin supply is fiat-backed centralized has different
risk characteristics than one with a diverse mix.

## Depeg Monitoring

Check if any stablecoin is trading off its peg.

### Step 1 -- Get prices for major stablecoins

```
defillama:get_token_prices
  token: ["coingecko:tether", "coingecko:usd-coin", "coingecko:dai", "coingecko:frax", "coingecko:ethena-usde"]
```

### Step 2 -- Flag deviations

Any stablecoin trading > 0.5% away from its peg ($1.00 for USD
stablecoins) warrants attention. > 2% deviation is a serious depeg event.

For historical depeg analysis, add `period: "30d"` to track price
stability over time. High price variance (even within the peg band)
signals lower confidence.

## Cross-Chain Stablecoin Bridge Flows

Understand how stablecoins move between chains.

### Step 1 -- Bridge flows

```
defillama:get_bridge_flows
  chain: "<chain>"
  period: "30d"
  sort_by: "volume desc"
```

### Step 2 -- Correlate with supply changes

Compare net bridge inflows with stablecoin supply changes on the same
chain over the same period. If supply is growing faster than bridge
inflows, the excess is native minting (USDC/USDT issued directly on
that chain). If bridge inflows exceed supply growth, capital is
arriving but being deployed into non-stablecoin assets.

## Examples

**Example 1:**
User: "What's the stablecoin breakdown on Arbitrum?"
Workflow: `get_stablecoin_supply(chain: "arbitrum", sort_by: "circulating_supply desc")` then compute dominance percentages.

**Example 2:**
User: "Which chains are gaining stablecoin supply?"
Workflow: `get_stablecoin_supply(sort_by: "circulating_supply desc", limit: 30)` for current, then `get_stablecoin_supply(period: "30d")` for historical. Aggregate by chain, compute deltas, sort by change.

**Example 3:**
User: "Is USDT too dominant on Tron?"
Workflow: `get_stablecoin_supply(chain: "tron", sort_by: "circulating_supply desc")`. Compute dominance. If USDT > 60%, flag concentration risk.

**Example 4:**
User: "Are any stablecoins depegging right now?"
Workflow: `get_token_prices(token: ["coingecko:tether", "coingecko:usd-coin", "coingecko:dai", "coingecko:frax"])`. Flag any price > 0.5% from $1.00.

**Example 5:**
User: "USDC vs USDT market share over the last 90 days"
Workflow: `get_stablecoin_supply(stablecoin: "coingecko:tether", period: "90d")` and `get_stablecoin_supply(stablecoin: "coingecko:usd-coin", period: "90d")` in parallel. Plot total circulating supply over time for each. Compute relative market share at each date.

**Example 6:**
User: "How much of Solana's DeFi depends on centralized stablecoins?"
Workflow: `get_stablecoin_supply(chain: "solana", sort_by: "circulating_supply desc")`. Classify each stablecoin by mechanism. Sum fiat-backed centralized supply and express as % of total. Cross-reference with `get_chain_metrics(chain: "solana")` TVL.
