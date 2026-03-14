---
name: btc
description: "Ask anything about Bitcoin companies. Examples: 'strategy.com', 'top 10 mining', 'usa', 'report etf', 'whales', 'Marathon', 'countries'"
---

# Bitcoin Companies

One command for all Bitcoin treasury data. Just say what you want.

## Step 1: Route the query

Classify the user's input into one of three intents:

| Check (in order) | Examples | Intent |
|-------------------|----------|--------|
| Contains a domain (has a `.` with no spaces) | `strategy.com`, `coinbase.com` | **LOOKUP** |
| Starts with "report" or "analysis" | `report mining`, `analysis etf` | **REPORT** |
| Matches a country name, code, or alias | `usa`, `united states`, `france`, `japan` | **LIST** (country) |
| Matches a category | `mining`, `etf`, `exchange`, `stocks`, `custody`, `software`, `private` | **LIST** (category) |
| Matches a tier name | `whale`, `shark`, `sovereign`, `shrimp`, `dolphin` | **LIST** (tier) |
| Contains "top N", "list", "rank", "leaderboard" | `top 10`, `list verified`, `rankings` | **LIST** |
| Contains "stats", "overview", "how much", "total" | `total btc held`, `stats`, `market overview` | **REPORT** |
| Contains "countries" or "tiers" | `list countries`, `show tiers` | **LIST** (special) |
| Contains "weather", "sentiment", "activity" | `weather for strategy.com`, `market sentiment` | **LOOKUP** + weather |
| Everything else (assumed company name) | `Marathon`, `Coinbase`, `MicroStrategy` | **LOOKUP** (search) |

**Country aliases to resolve:**

| Alias | Resolves to |
|-------|-------------|
| `usa`, `us` | `united-states` |
| `uk` | `united-kingdom` |

---

## LOOKUP intent

Look up a specific company by domain or search by name.

### Fetch

**If input contains a dot** (domain):
```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
```

**Otherwise** (name search):
```
WebFetch https://bitcoincompanies.co/api/v1/companies?q={query}&limit=5
```

Also fetch price for USD conversion:
```
WebFetch https://bitcoincompanies.co/api/v1/price
```

If user asked about weather/activity/sentiment, also fetch:
```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/weather
```

### Format

**Single company** (domain lookup or single search result):
```
**{name}** ({ticker}) — {tier.emoji} {tier.name}
{btc} BTC (${btc * price} USD) | Verified: {verified_percentage}%
Rank #{rank} | {country.name}
Supply: {supply_percentage}% of 21M
Profile: https://bitcoincompanies.co/{domain}
```

**Multiple results** (name search):

| # | Company | Ticker | BTC | Tier | Country |
|---|---------|--------|-----|------|---------|
| 1 | ... | ... | ... | ... | ... |

---

## LIST intent

Display the leaderboard with optional filters.

### Parse filters

Combine multiple filters from the query:

| User says | API param |
|-----------|-----------|
| `top 10` / `top 20` | `limit=10` / `limit=20` |
| `mining` / `miners` | `category=mining` |
| `etf` / `etfs` | `category=etf` |
| `exchange` / `exchanges` | `category=exchange` |
| `stocks` / `public` | `category=stocks` |
| `custody` | `category=custody` |
| `software` | `category=software` |
| `private` | `category=private` |
| `usa` / `united states` | `country=united-states` |
| `france` | `country=france` |
| `japan` | `country=japan` |
| Any country name | `country={slug}` |
| `whale` / `whales` | `tier=whale` |
| `sovereign` | `tier=sovereign` |
| Any tier name | `tier={tier}` |
| `verified` / `verified only` | `only_verified=true` |
| `page 2` | `page=2` |

Default: `limit=25` with no filters.

### Fetch

```
WebFetch https://bitcoincompanies.co/api/v1/companies?{params}
```

**Special cases:**

If user asks "list countries" or "countries by BTC":
```
WebFetch https://bitcoincompanies.co/api/v1/countries?limit=20
```

If user asks "list tiers" or "what are the tiers":
```
WebFetch https://bitcoincompanies.co/api/v1/tiers
```

### Format

```markdown
## Bitcoin Treasury Leaderboard {filter description}
{total_count} companies | Page {page}

| # | Company | Ticker | BTC | Tier | Verified | Country |
|---|---------|--------|-----|------|----------|---------|
| 1 | Strategy | MSTR | 500,000 | 👑 Sovereign | 100% | US |
| ... |

{has_more ? "Page {page} of {total_pages}. Say 'page 2' to see more." : ""}
```

Use `tier.emoji` from the API response. Format BTC with comma separators.

---

## REPORT intent

Generate an aggregate treasury analysis.

### Fetch (in parallel)

Always:
```
WebFetch https://bitcoincompanies.co/api/v1/stats
WebFetch https://bitcoincompanies.co/api/v1/price
```

Top companies (with any filter):
```
WebFetch https://bitcoincompanies.co/api/v1/companies?limit=10{&category or &country or &tier}
```

Optionally (if user asks about sentiment/weather):
```
WebFetch https://bitcoincompanies.co/api/v1/weather
```

### Format

```markdown
# Bitcoin Treasury Report {filter if any}

## Market
BTC Price: ${price} | 24h: {change_24h}%

## Summary
- Total companies: {total_companies}
- Total BTC held: {total_btc} ({supply_percentage}% of 21M)
- Verified: {total_verified_btc} BTC ({verified_percentage}%)
- Total USD value: ${total_btc * price}

## Top 10 {filter if any}
| # | Company | Ticker | BTC | Tier | Verified |
|---|---------|--------|-----|------|----------|
| 1 | Strategy | MSTR | 500,000 | 👑 Sovereign | 100% |
| ... |

## By Category
| Category | Companies | BTC | % of Total |
|----------|-----------|-----|------------|
| ... from stats.by_category |

## By Tier
| Tier | Companies | BTC |
|------|-----------|-----|
| ... from stats.by_tier |
```

---

## API Reference

```
Base: https://bitcoincompanies.co/api/v1

GET /companies                        Paginated leaderboard
  ?q={query}                          Search by name/ticker/domain
  ?category=mining                    Filter: mining, etf, exchange, stocks, custody, software, private
  ?tier=whale                         Filter: sovereign, leviathan, humpback, whale, shark, dolphin, fish, octopus, crab, shrimp
  ?country=united-states              Filter by country slug
  ?sort=rank|verified|best_reviewed   Sort order
  ?only_verified=true                 Verified companies only
  ?limit=25&page=1                    Pagination (max 100)

GET /companies/{domain}               Company detail (by domain)
GET /companies/{domain}/weather       Company market weather
GET /companies/{domain}/claims        Company claims history

GET /stats                            Aggregate stats (totals, by_category, by_tier)
GET /price                            Current BTC price + 24h change
GET /weather                          Global market sentiment
GET /countries?limit=20               Countries ranked by BTC
GET /tiers                            Tier definitions with ranges
```

## Response formats

**Company detail:**
- `id` (domain), `name`, `ticker`, `category`, `logo_url`
- `rank` (integer), `tier` (`{ id, name, emoji }`)
- `treasury` (`{ btc, verified_btc, claimed_btc, verified_percentage, supply_percentage }`)
- `country` (`{ id, code, name }`)
- `reviews` (`{ count, average_rating }`)
- `profile_url`, `updated_at`

**Company list item:**
- `id`, `name`, `ticker`, `category`, `logo_url`
- `rank`, `tier` (`{ id, name, emoji }`)
- `btc`, `verified_percentage`, `supply_percentage`
- `country` (`{ id, code, name }`)
- `reviews_count`, `reviews_average`, `updated_at`
