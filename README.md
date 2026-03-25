# DefiLlama Skills

Agent skills for [DefiLlama MCP](https://mcp.defillama.com) — 23 DeFi analytics tools with guided workflows.

## Quick Install

Paste this prompt into your AI agent:

```
Read https://raw.githubusercontent.com/DefiLlama/defillama-skills/master/defillama-setup/SKILL.md and follow the instructions to install DefiLlama MCP.
```

## Manual Setup

### 1. Add the MCP server

**Claude Code:**
```bash
claude mcp add defillama --transport http https://mcp.defillama.com/mcp
```

**OpenClaw** (`~/.openclaw/openclaw.json`):
```json
{
  "mcpServers": {
    "defillama": {
      "url": "https://mcp.defillama.com/mcp"
    }
  }
}
```

**Cursor / Windsurf / Claude Desktop:**
```json
{
  "mcpServers": {
    "defillama": {
      "url": "https://mcp.defillama.com/mcp"
    }
  }
}
```

### 2. Log in

On first connection your browser opens the DefiLlama login page. Sign in with your account — requires an [active API subscription](https://defillama.com/subscribe). That's it. No API keys, no env vars.

### 3. Install skills (optional)

Skills teach your agent structured DeFi analysis workflows. Copy them into your agent's skills directory:

**OpenClaw:**
```bash
git clone https://github.com/DefiLlama/defillama-skills /tmp/defillama-skills
cp -r /tmp/defillama-skills/*/ ~/.openclaw/skills/
```

**Claude Code:**
```bash
git clone https://github.com/DefiLlama/defillama-skills /tmp/defillama-skills
cp -r /tmp/defillama-skills/*/ .claude/skills/
```

## Skills

| Skill | Description |
|-------|-------------|
| **defi-data** | Core reference — maps any DeFi question to the right tool and params |
| **defi-market-overview** | Full market snapshot: TVL, categories, chains, events, stablecoins, ETFs |
| **protocol-deep-dive** | Complete protocol report: TVL, fees, yields, income, users, token |
| **token-research** | Token analysis: price, unlocks, DeFi deposits, yield opportunities |
| **chain-ecosystem** | Blockchain overview: TVL, top protocols, bridges, stablecoins, users |
| **market-analysis** | Screening and comparison: valuation ratios, growth, cross-entity |
| **yield-strategies** | Yield hunting: pool filtering, APY conventions, capacity assessment |
| **risk-assessment** | Risk evaluation: hacks, oracles, treasury, fundamentals, yield flags |
| **flows-and-events** | Capital flows: bridges, ETFs, stablecoins, hacks, raises, OI |
| **institutional-crypto** | Institutional exposure: corporate holdings, ETF flows, mNAV ratios |

## Tools (23)

| Tool | Description |
|------|-------------|
| `resolve_entity` | Fuzzy-match protocol, chain, or token names to exact slugs |
| `get_market_totals` | Global DeFi TVL, DEX volume, derivatives volume |
| `get_protocol_metrics` | Protocol TVL, fees, revenue, mcap, ratios, trends |
| `get_protocol_info` | Protocol metadata, URLs, audit info, tags |
| `get_chain_metrics` | Chain TVL, gas fees, revenue, DEX volume |
| `get_chain_info` | Chain metadata, type, L2 parent |
| `get_category_metrics` | Category rankings by TVL, fees, protocol count |
| `list_categories` | List all valid categories |
| `get_token_prices` | Token price, mcap, volume, ATH |
| `get_token_tvl` | Token deposits across DeFi protocols |
| `get_token_unlocks` | Vesting schedules and upcoming unlocks |
| `get_yield_pools` | Pool APY, TVL, lending/borrowing rates |
| `get_stablecoin_supply` | Stablecoin issuance by chain |
| `get_bridge_flows` | Bridge volume and net flows by chain |
| `get_etf_flows` | Bitcoin and Ethereum ETF inflows/outflows |
| `get_dat_holdings` | Institutional crypto holdings and mNAV |
| `get_events` | Hacks, fundraises, protocol events |
| `get_oracle_metrics` | Oracle TVS and protocol coverage |
| `get_cex_volumes` | Centralized exchange trading volume |
| `get_open_interest` | Derivatives open interest |
| `get_treasury` | Protocol treasury holdings |
| `get_user_activity` | Daily active users and transactions |
| `get_income_statement` | Protocol revenue breakdown |
