---
name: defi-due-diligence
description: >
  Structured protocol evaluation framework for DeFi allocation decisions.
  Scores protocols across five dimensions: revenue quality, yield
  sustainability, security posture, competitive positioning, and token
  value capture. Composes multiple tools into a scored assessment with
  explicit bull/bear signals. Use when users ask whether to allocate to a
  protocol, evaluate protocol fundamentals, assess DeFi investment risk,
  or need a structured opinion beyond raw data.
---

# DeFi Due Diligence

Evaluate a protocol's fitness for capital allocation by scoring five
dimensions. Each dimension uses specific tools and produces a signal
(bullish, neutral, bearish) with supporting evidence.

This skill goes beyond `protocol-deep-dive` (which reports data) by
applying judgment frameworks that flag what matters for allocation
decisions.

## Workflow

### Step 1 -- Resolve and fetch core metrics

```
defillama:resolve_entity
  entity_type: "protocol"
  name: "<user-provided name>"
```

```
defillama:get_protocol_metrics
  protocol: "<slug>"
  metrics: ["tvl_base", "fees_1d", "revenue_1d", "holder_revenue_1d",
            "mcap", "fdv_outstanding", "ps_ratio", "pf_ratio",
            "tvl_base_7d_pct_change", "tvl_base_30d_pct_change",
            "fees_1d_30d_pct_change", "revenue_1d_30d_pct_change"]
```

```
defillama:get_protocol_info
  protocol: "<slug>"
```

Use `get_protocol_info` to get category, chains, audit status, and age.

### Step 2 -- Score each dimension

---

#### Dimension 1: Revenue Quality

**Question:** Is the protocol generating real, sustainable revenue?

**Data needed:** `revenue_1d`, `fees_1d`, `holder_revenue_1d`,
`revenue_1d_30d_pct_change` from Step 1.

Fetch income breakdown:

```
defillama:get_income_statement
  protocol: "<slug>"
```

**Scoring criteria:**

- Revenue/fees ratio (protocol take rate): > 20% is healthy, < 5% means
  almost all fees go to LPs/stakers and the protocol captures little.
- `holder_revenue` > 0 means the protocol shares revenue with token
  holders (buybacks, dividends). Strong value capture signal.
- Revenue growing (`revenue_1d_30d_pct_change` > 0) while TVL is flat or
  growing: protocol is becoming more capital-efficient.
- Revenue declining while TVL grows: the protocol is subsidizing growth
  with incentives. Check yield sustainability (Dimension 2).

**Signals:**
- Bullish: take rate > 20%, holder revenue > 0, revenue growing
- Neutral: take rate 10-20%, no holder revenue, stable revenue
- Bearish: take rate < 5%, revenue declining, no path to holder revenue

---

#### Dimension 2: Yield Sustainability

**Question:** Are the yields on this protocol organic or inflationary?

```
defillama:get_yield_pools
  protocol: "<slug>"
  include_borrow: true
  sort_by: "tvl desc"
  limit: 10
```

**Scoring criteria:**

- Compare `apy_base` (organic, from trading fees or interest) vs
  `apy_reward` (token incentives). If `apy_reward > apy_base` for the
  majority of pools, yields are incentive-driven and will compress when
  emissions decline.
- Check `apy_base` stability using `include_volatility: true`:

```
defillama:get_yield_pools
  protocol: "<slug>"
  include_volatility: true
  sort_by: "tvl desc"
  limit: 10
```

High `cv_30d` (coefficient of variation > 0.5) means APY is unstable.

- For lending protocols: compare supply APY with borrow APY. Healthy
  utilization = borrow rate moderately above supply rate. If borrow
  rates are extremely high, borrowers may leave.

**Signals:**
- Bullish: apy_base > apy_reward across top pools, low APY volatility
- Neutral: mixed base/reward split, moderate volatility
- Bearish: apy_reward dominates, high volatility, declining APYs

---

#### Dimension 3: Security Posture

**Question:** What is the protocol's hack history, oracle dependency, and
treasury resilience?

```
defillama:get_events
  protocol: "<slug>"
  event_type: "hacks"
```

```
defillama:get_oracle_metrics
  oracle: null
```

Use oracle metrics to check which oracle the protocol depends on.
Cross-reference with `get_protocol_info` audit status.

```
defillama:get_treasury
  treasury: "<slug>"
```

**Scoring criteria:**

- Hack history: any hack > 10% of current TVL is a major flag. Multiple
  hacks across different attack vectors suggest systemic issues.
  Recovery (high `returned_funds`) partially mitigates.
- Oracle dependency: single-oracle protocols have concentration risk.
  Chainlink dependency is standard; no oracle or custom oracle warrants
  scrutiny.
- Audit status: audited > unaudited. Multiple auditors > single auditor.
  Recent audit > stale audit.
- Treasury: `treasury_excl_own_token` > 12 months of operating costs
  (estimate from revenue run rate) means the protocol can survive a
  sustained downturn. Treasury dominated by own token is fragile
  (token price crash = treasury evaporates).

**Signals:**
- Bullish: no hacks, audited, diversified treasury, standard oracle
- Neutral: minor hack with recovery, audited, adequate treasury
- Bearish: major unrecovered hack, unaudited, treasury < 6 months
  runway or dominated by own token

---

#### Dimension 4: Competitive Positioning

**Question:** How does this protocol compare to others in its category?

```
defillama:get_category_metrics
  category: "<protocol's category>"
  metrics: ["tvl_base", "fees_1d", "revenue_1d", "protocol_count"]
```

```
defillama:get_protocol_metrics
  protocol: ["<slug>", "<competitor_1>", "<competitor_2>"]
  metrics: ["tvl_base", "fees_1d", "revenue_1d", "ps_ratio",
            "tvl_base_30d_pct_change"]
```

Identify competitors from the same category. For lending: Aave,
Compound, Morpho, Spark. For DEXs: Uniswap, Curve, Aerodrome.
Use `get_protocol_info(category: "<category>")` to discover top
protocols in the category if needed.

**Scoring criteria:**

- Market share: protocol TVL / category TVL. > 30% is dominant.
  < 5% is a small player.
- Relative valuation: compare `ps_ratio` to category peers. Significantly
  lower P/S with similar or better growth = potentially undervalued.
  Significantly higher P/S with slower growth = potentially overvalued.
- Growth vs peers: if the protocol's `tvl_base_30d_pct_change` is positive
  while peers are flat or negative, it's gaining share. The reverse means
  it's losing ground.
- Multi-chain presence: protocols on more chains have a larger addressable
  market but also more attack surface.

**Signals:**
- Bullish: gaining market share, lower P/S than peers, multi-chain
- Neutral: stable share, in-line valuation
- Bearish: losing share, premium valuation, single-chain

---

#### Dimension 5: Token Value Capture

**Question:** Does the token benefit from protocol success?

```
defillama:get_token_prices
  token: "coingecko:<token_id>"
```

```
defillama:get_token_unlocks
  token: "coingecko:<token_id>"
  query_type: "window"
```

**Scoring criteria:**

- `holder_revenue > 0` (from Dimension 1): direct value flow to holders.
- Upcoming unlock schedule: large unlocks (> 5% of circulating supply)
  within 90 days create sell pressure.
- FDV / mcap ratio: > 3x means significant dilution ahead.
  `fdv_outstanding` from protocol metrics gives a more realistic dilution
  estimate than raw FDV.
- Token utility: governance-only tokens with no revenue share or
  buyback mechanism have weaker value capture than tokens with
  fee switches, buybacks, or staking yields.

**Signals:**
- Bullish: revenue sharing active, no major unlocks, FDV/mcap < 2x
- Neutral: governance-only but buyback proposed, moderate unlock schedule
- Bearish: no value capture, large imminent unlocks, FDV/mcap > 5x

### Step 3 -- Synthesize

Present the five dimensions as a summary table:

| Dimension | Signal | Key Evidence |
|-----------|--------|-------------|
| Revenue Quality | Bullish | 25% take rate, holder rev active, growing |
| Yield Sustainability | Neutral | Mixed base/reward, moderate volatility |
| Security Posture | Bullish | No hacks, audited, 18mo treasury runway |
| Competitive Position | Bullish | Gaining share, lower P/S than peers |
| Token Value Capture | Bearish | No fee switch, 15% unlock in 60 days |

Then provide a 2-3 sentence overall assessment that weighs the signals.
A single bearish dimension does not make the protocol uninvestable. But
two or more bearish signals in revenue quality + yield sustainability +
security is a strong caution.

**Do not give investment advice.** Frame assessments as "the data
suggests" or "signals indicate" rather than "you should buy/sell."
Present the framework and let the user make the decision.

## Examples

**Example 1:**
User: "Should I provide liquidity to Aave?"
Workflow: Full five-dimension evaluation. Resolve "aave", fetch all
metrics, score each dimension, present summary table.

**Example 2:**
User: "Is Pendle a good investment?"
Workflow: Full evaluation. Pay special attention to Dimension 2 (yield
sustainability) since Pendle's model is yield-centric, and Dimension 5
(token value capture) since PENDLE tokenomics are central to the thesis.

**Example 3:**
User: "Compare Aave and Morpho for lending allocation"
Workflow: Run Dimensions 1-3 for both protocols in parallel. Skip
Dimensions 4-5 since this is a deployment decision, not a token
investment. Focus on revenue quality (which protocol captures more
value), yield sustainability (which has better organic rates), and
security (hack history, audit status).

**Example 4:**
User: "Is this new protocol safe to use?"
Workflow: Focus on Dimension 3 (security posture). Check hack history,
audit status, oracle dependency, treasury size. If the protocol has
< 30 days of history, flag the limited track record as a risk factor
regardless of other signals.

**Example 5:**
User: "What's the bear case for Lido?"
Workflow: Run all five dimensions but emphasize bearish signals.
Check staking concentration risk (Lido's share of total ETH staked),
regulatory exposure (liquid staking regulatory scrutiny), and token
value capture (LDO utility). Present the bearish evidence explicitly
while noting any mitigating bullish signals.
