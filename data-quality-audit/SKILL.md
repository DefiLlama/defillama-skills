---
name: data-quality-audit
description: >
  Data-quality and anomaly triage for DefiLlama metrics. Use when the user asks
  to validate suspicious data, investigate metric spikes or drops, find bad
  TVL/fee/revenue/yield values, compare against peers, or prepare a
  reproducible bug report for DefiLlama data.
---

# DefiLlama Data Quality Audit

Audit suspicious DeFi metrics by combining current values, history, peer
context, and related dimensions. The goal is not to prove data is wrong from a
single outlier; the goal is to produce a reproducible triage report that shows
what changed, why it looks suspicious, and what to verify next.

## Workflow

### Step 1 - Define the audit target

Identify the entity type and metric family before querying:

| User asks about | Primary tool |
|-----------------|--------------|
| Protocol TVL, fees, revenue, mcap | `defillama:get_protocol_metrics` |
| Chain TVL, gas fees, app revenue | `defillama:get_chain_metrics` |
| Category aggregate metrics | `defillama:get_category_metrics` |
| Yield pool APY / TVL | `defillama:get_yield_pools` |
| Stablecoin supply | `defillama:get_stablecoin_supply` |
| Bridge inflows/outflows | `defillama:get_bridge_flows` |
| Events, hacks, raises | `defillama:get_events` |
| Token price / mcap / volume | `defillama:get_token_prices` |

Resolve ambiguous names first:

```
defillama:resolve_entity
  name: "<user-provided name>"
```

Use parent protocol slugs for aggregate protocol metrics unless the user asks
about a specific sub-protocol version.

### Step 2 - Pull current value and history

For metric anomalies, fetch both the current snapshot and a recent history
window. Use `30d` by default; use `90d` when the user asks whether a move is
structural.

Example protocol audit:

```
defillama:get_protocol_metrics
  protocol: "<slug>"
  metrics: ["tvl_base", "fees_1d", "revenue_1d", "tvl_base_7d_pct_change", "tvl_base_30d_pct_change"]

defillama:get_protocol_metrics
  protocol: "<slug>"
  metrics: ["tvl_base", "fees_1d", "revenue_1d"]
  period: "30d"
```

Example yield audit:

```
defillama:get_yield_pools
  protocol: "<slug>"
  include_borrow: true
  include_volatility: true
  sort_by: "apy desc"
```

### Step 3 - Run sanity checks

Check these before calling anything wrong:

- **Null vs zero**: `NULL` means unavailable, not zero. Treat new nulls as data
  quality issues only if nearby related data still exists.
- **Stock vs flow**: do not sum TVL, price, mcap, or supply across dates. Flows
  such as fees, revenue, and volume can be summed over periods.
- **Percent conventions**: `*_pct_change` values are decimals (`0.25` = 25%).
  APY values are already percentages (`5.2` = 5.2%).
- **Fee relationship**: revenue should usually be less than or equal to fees.
  If revenue exceeds fees, flag for manual review unless the tool definition
  explicitly explains the difference.
- **Negative values**: negative TVL, mcap, supply, APY base, or volume usually
  indicates bad source data or a transformation issue. Signed flow fields such
  as ETF flow and bridge inflow can legitimately be negative.
- **Dust moves**: ignore percentage spikes on tiny absolute values unless the
  user is auditing small protocols specifically.
- **FDV**: avoid FDV as a primary anomaly signal. Total supply data can be
  unreliable; use mcap, revenue, fees, and TVL first.
- **Liquid staking / double count**: do not add TVL components twice. Default
  TVL is `tvl_base`.

### Step 4 - Compare against peers

Outliers are stronger when peer context disagrees. Compare the target with
similar protocols, its category, or its chain.

```
defillama:get_protocol_metrics
  protocol: ["<target>", "<peer-1>", "<peer-2>"]
  metrics: ["tvl_base", "fees_1d", "revenue_1d", "ps_ratio", "pf_ratio"]
```

For unknown peers, use category rankings:

```
defillama:get_category_metrics
  category: "<category>"
  metrics: ["tvl_base", "fees_1d", "revenue_1d", "protocol_count"]
  sort_by: "tvl_base desc"
  limit: 20
```

### Step 5 - Classify severity

Use this severity scale:

| Severity | When to use |
|----------|-------------|
| Critical | Impossible values, broken extraction, or a large move that changes rankings/material conclusions |
| High | Large unexplained divergence from history or peers, stale-looking data, revenue/fee inconsistency |
| Medium | Suspicious but plausible outlier, missing secondary metric, unusually high APY with low TVL |
| Low | Minor formatting, small absolute impact, or noisy low-liquidity metric |

### Step 6 - Produce a reproducible report

Always include:

1. **Summary**: one sentence with severity and suspected issue.
2. **Evidence table**: metric, current value, historical comparison, peer/category context.
3. **Reproducer**: exact MCP tool calls and key params used.
4. **Likely cause**: stale source, null/zero confusion, unit mismatch, duplicate counting, methodology difference, or unknown.
5. **Next check**: the primary source URL, protocol docs, adapter/source code, or maintainer question needed to confirm.

Do not state "DefiLlama is wrong" unless you have a direct source-of-truth
comparison. Prefer "needs review" when the evidence is only internal
inconsistency or statistical outlier detection.

## Quick Audit Recipes

### TVL spike/drop

1. Current protocol metrics with `tvl_base`, `tvl_base_7d_pct_change`,
   `tvl_base_30d_pct_change`.
2. Historical `period: "30d"` for `tvl_base`.
3. Category peers sorted by TVL.
4. Protocol events and hacks over the same window.

### Fee or revenue anomaly

1. Current `fees_1d`, `revenue_1d`, `holder_revenue_1d`.
2. Historical `period: "30d"` for the same metrics.
3. Compare revenue to fees and peer revenue/TVL ratios.
4. If revenue exceeds fees, label as high severity unless explained.

### Yield outlier

1. Query pool rows with `include_volatility: true` and `include_borrow: true`.
2. Check `apy`, `apy_base`, `apy_reward`, `tvl`, and volatility fields.
3. High APY with low TVL is a warning, not proof of bad data.
4. High `apy_reward` means emissions-driven yield; do not call it real yield.

### Stablecoin or bridge flow anomaly

1. Pull stablecoin supply or bridge flow for the target chain and period.
2. Compare against chain TVL and recent protocol events.
3. Signed flow fields can be negative; supply and TVL should not be.
