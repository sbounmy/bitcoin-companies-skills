---
name: bitcoin-report
description: Generate a Bitcoin treasury analysis report with stats, rankings, and tier breakdowns. Use when the user wants aggregate analysis, category reports, or market overview.
---

# Bitcoin Report

Generate a structured treasury analysis report by fetching data from the Bitcoin Companies API.

## How to use

The user may provide a **filter** (category, country, or tier) or ask for a general report.

### Step 1: Fetch data (3 calls in parallel)

Always fetch these:
```
WebFetch https://bitcoincompanies.co/api/v1/stats
WebFetch https://bitcoincompanies.co/api/v1/price
```

Then fetch top companies with any filter the user specified:
```
WebFetch https://bitcoincompanies.co/api/v1/companies?limit=10&category={filter}
WebFetch https://bitcoincompanies.co/api/v1/companies?limit=10&country={filter}
```

If no filter, fetch the overall top 10:
```
WebFetch https://bitcoincompanies.co/api/v1/companies?limit=10
```

Optionally fetch global weather for market sentiment:
```
WebFetch https://bitcoincompanies.co/api/v1/weather
```

### Step 2: Format the report

Structure the output as:

```markdown
# Bitcoin Treasury Report {category/country if filtered}

## Market
BTC Price: ${price} | 24h: {change_24h}%

## Summary
- Total companies: {total_companies}
- Total BTC held: {total_btc} ({supply_percentage}% of 21M)
- Verified: {total_verified_btc} BTC ({verified_percentage}%)
- Total USD value: ${total_btc * price}

## Top 10 {category if filtered}
| # | Company | Ticker | BTC | Tier | Verified |
|---|---------|--------|-----|------|----------|
| 1 | Strategy | MSTR | 500,000 | Sovereign | 100% |
| ... | ... | ... | ... | ... | ... |

## By Category
| Category | Companies | BTC | % of Total |
|----------|-----------|-----|------------|
| ... from stats.by_category ... |

## By Tier
| Tier | Companies | BTC |
|------|-----------|-----|
| ... from stats.by_tier ... |
```

### Common filters

| User says | API param |
|-----------|-----------|
| "mining report" | `?category=mining` |
| "ETF report" | `?category=etf` |
| "US companies" | `?country=united-states` |
| "whale tier" | `?tier=whale` |
| "verified only" | `?only_verified=true` |

## API Reference

```
Base: https://bitcoincompanies.co/api/v1

GET /stats                          Aggregate stats (total_companies, total_btc, by_category, by_tier)
GET /price                          Current BTC price + 24h change
GET /companies?category=mining      Filtered leaderboard
GET /companies?country=united-states  By country
GET /companies?tier=whale           By tier
GET /weather                        Global market sentiment
```
