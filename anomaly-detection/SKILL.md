---
name: anomaly-detection
description: >
  Forensic analysis of TVL, fee, revenue, and price anomalies across
  protocols, chains, and stablecoins. Combines pct_change windows with
  root-cause filters (hacks, token migrations, market-wide moves, native
  token re-pricing) to separate real anomalies from artifacts. Use when
  the user asks "why did X spike", "anything weird with Y", "scan for
  outliers today", or wants pre-trade sanity checks before acting on a
  TVL or fee move.
---

# Anomaly Detection

Catch and explain spikes or drops in protocol TVL, fees, revenue, user
activity, stablecoin supply, and bridge flows. The goal is to flag what
moved, then rule out benign causes before calling it a real anomaly.

This skill is forensic (what just happened, why), not predictive. For
forward-looking risk, use `risk-assessment`.

## Workflow

### Step 1 - Set the scope

Two scan modes:

- **Targeted**: user names a protocol, chain, or stablecoin. Skip to Step 2.
- **Broad scan**: user asks "anything weird today" or "outliers". Pull the
  largest movers across the universe in Step 2 first, then drill in.

If the entity is named but ambiguous, resolve it.

```
defillama:resolve_entity
  entity_type: "protocol"
  name: "<user-provided name>"
```

### Step 2 - Pull the baseline with pct_change windows

For a single protocol:

```
defillama:get_protocol_metrics
  protocol: "<slug>"
```

Read these fields: `tvl_base_24h_pct_change`, `tvl_base_7d_pct_change`,
`tvl_base_30d_pct_change`, `fees_24h_pct_change`, `fees_7d_pct_change`,
`revenue_24h_pct_change`.

For a broad scan of biggest movers:

```
defillama:get_protocol_metrics
  sort_by: "tvl_base_24h_pct_change desc"
  limit: 20
```

Then again with `asc` to surface the largest drops. Repeat with
`fees_24h_pct_change` to catch fee anomalies that don't yet show in TVL.

### Step 3 - Apply severity bands

Score each candidate. The bands are investigation triggers, not final
verdicts: anything flagged here still has to survive Step 4.

| Magnitude | 24h |Δ| | 7d |Δ| | Severity |
|-----------|---------|--------|----------|
| Small | 15 to 30% | 30 to 60% | LOW |
| Large | 30 to 60% | 60 to 120% | MEDIUM |
| Extreme | over 60% | over 120% | HIGH |

Drop candidates that fail either gate:

- Absolute TVL under $1M (noise floor, % change is meaningless)
- Protocol age under 30 days (TVL ramp from launch is not an anomaly)

For age, check listing history with `get_protocol_info`.

### Step 4 - Rule out benign causes

Run these in parallel for each flagged candidate. Any positive match
downgrades or removes the anomaly flag.

**Hack or exploit:**

```
defillama:get_events
  protocol: "<slug>"
  event_type: "hacks"
  period: "7d"
```

A hack within the move window explains the drop. Tag as `EXPLAINED: hack`.

**Token migration or relisting:**

```
defillama:get_protocol_info
  protocol: "<slug>"
```

Read the metadata for fork-of, parent, or audit notes mentioning a
migration. TVL can appear to "spike" when a v2 launches and the v1
deprecates; the aggregate is unchanged.

**Market-wide move:**

```
defillama:get_chain_metrics
  chain: "<protocol's primary chain>"
```

If the chain itself moved with similar magnitude, this is not
protocol-specific. Compare `tvl_24h_pct_change` on the chain vs the
protocol. Tag as `EXPLAINED: market-wide`.

**Native token re-pricing:**

```
defillama:get_token_prices
  token: "coingecko:<protocol's native token>"
```

TVL is denominated in USD. If the protocol holds a meaningful share of
its TVL in its own token (common for staking and ve-models), a token
price move leaks directly into TVL. Cross-check with
`tvl_base_excl_own_tokens` if available.

**Category-level move:**

```
defillama:get_category_metrics
  category: "<protocol's category>"
  period: "24h"
```

If the entire category moved, the signal is sector rotation, not a
single-protocol anomaly.

### Step 5 - Stablecoin and depeg anomalies

For stablecoins, use a different toolset.

**Supply contraction (depeg precursor):**

```
defillama:get_stablecoin_supply
  stablecoin: "<slug>"
  period: "7d"
```

A circulating supply drop over 5% in 7 days is a redemption signal.

**Current peg:**

```
defillama:get_token_prices
  token: "coingecko:<stablecoin>"
```

Compute deviation from $1.00 in basis points. Anything beyond 25 bps
warrants escalation; beyond 100 bps is HIGH severity.

**Exit liquidity check:**

```
defillama:get_yield_pools
  symbol: "<stablecoin>"
  sort_by: "apy desc"
  limit: 5
```

Sudden lending APY spikes on the stablecoin signal holders paying up to
borrow against and exit it.

### Step 6 - Bridge flow anomalies

Capital movement across chains often precedes or accompanies anomalies.

```
defillama:get_bridge_flows
  chain: "<chain>"
  period: "24h"
  sort_by: "inflows desc"
```

Large net inflows on a chain that just had a TVL spike point to fresh
capital arrival; large net outflows on a chain with a TVL drop point to
exit. Pair with the protocol-level finding.

### Step 7 - Synthesize

For each candidate, output one of:

- `ANOMALY` with severity, magnitude, and probable cause
- `EXPLAINED` with the rule that explains it (hack, migration, market,
  token-price, category)
- `INSUFFICIENT_DATA` with the missing input

## Output Format

Present the report with these sections:

1. **Scan Summary** - one line per flagged candidate: protocol, magnitude,
   severity, classification.
2. **Confirmed Anomalies** - per anomaly: what moved, time window,
   probable cause, supporting data points (chain TVL, token price,
   category move).
3. **Explained Movers** - candidates that hit raw thresholds but were
   ruled out, with the rule that filtered them. This transparency is
   important; never report "no anomalies" without showing the filters.
4. **Pending Investigation** - candidates with insufficient data, with
   the specific tool call that would resolve the ambiguity.

## Tips

- TVL spikes from near-zero are nearly always new listings. Always check
  protocol age in Step 3 before flagging.
- A protocol with high token-concentration TVL (over 30% in its own
  token) will move with its token price. Use `tvl_base_excl_own_tokens`
  when comparing time series for these protocols.
- Bridge inflow on chain A plus protocol TVL outflow on chain A often
  indicates capital rotation between protocols, not loss of capital.
- Always show your filter chain. A user acting on an anomaly report
  needs to know what was ruled out and why.
- For fast-moving events, daily granularity can miss the spike. If
  available, request `period: "1h"` for the move window before drawing
  conclusions.
- Cross-reference with `get_events` event_type `protocol_events` for
  governance changes, parameter updates, or vault redeployments that
  would explain a TVL move without a hack.
- The native token re-pricing check is the most commonly missed
  filter. Many "TVL anomalies" are actually token price moves leaking
  through the USD denomination.
