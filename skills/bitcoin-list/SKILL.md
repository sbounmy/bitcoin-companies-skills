---
name: bitcoin-list
description: Display the Bitcoin company leaderboard as a formatted table. Use when the user wants to see rankings, top N lists, or filtered company listings.
---

# Bitcoin List

Fetch and display the Bitcoin treasury leaderboard from the Bitcoin Companies API.

## How to use

The user may specify filters like "top 10", "mining", "USA", "whales", or combine them.

### Step 1: Parse the request

Map user intent to API params:

| User says | Param |
|-----------|-------|
| "top 10" / "top 20" | `limit=10` / `limit=20` |
| "mining" / "miners" | `category=mining` |
| "etf" / "etfs" | `category=etf` |
| "exchange" / "exchanges" | `category=exchange` |
| "stocks" / "public" | `category=stocks` |
| "usa" / "united states" | `country=united-states` |
| "france" | `country=france` |
| "whale" / "whales" | `tier=whale` |
| "sovereign" | `tier=sovereign` |
| "verified" / "verified only" | `only_verified=true` |
| "page 2" | `page=2` |

Default: `limit=25` (no filters).

### Step 2: Fetch

```
WebFetch https://bitcoincompanies.co/api/v1/companies?{params}
```

### Step 3: Format as table

```markdown
## Bitcoin Treasury Leaderboard {filter description}
{total_count} companies | Page {page}

| # | Company | Ticker | BTC | Tier | Verified | Country |
|---|---------|--------|-----|------|----------|---------|
| 1 | Strategy | MSTR | 500,000 | Sovereign | 100% | US |
| 2 | Marathon Digital | MARA | 45,000 | Humpback | 85% | US |
| ... | ... | ... | ... | ... | ... | ... |

{has_more ? "More results available. Use `/bitcoin-list page 2`" : ""}
```

Use tier emoji from the `tier.emoji` field. Format BTC with comma separators.

### Step 4: Country or tier listing (alternative)

If user asks "list countries" or "countries by BTC":
```
WebFetch https://bitcoincompanies.co/api/v1/countries?limit=20
```

Format as country table with BTC totals.

If user asks "list tiers" or "what are the tiers":
```
WebFetch https://bitcoincompanies.co/api/v1/tiers
```

Format as tier definition table with ranges.

## API Reference

```
Base: https://bitcoincompanies.co/api/v1

GET /companies                      Paginated leaderboard
  ?category=mining                  Filter by category
  ?tier=whale                       Filter by tier
  ?country=united-states            Filter by country
  ?q=Marathon                       Search by name/ticker
  ?sort=rank|verified|best_reviewed Sort order
  ?only_verified=true               Verified companies only
  ?limit=25&page=1                  Pagination (max 100)

GET /countries?limit=20             Countries ranked by BTC
GET /tiers                          Tier definitions
```

## Response format

Each company in `data[]` has:
- `id` (domain), `name`, `ticker`, `category`, `logo_url`
- `rank` (integer), `tier` (`{ id, name, emoji }`)
- `btc`, `verified_percentage`, `supply_percentage`
- `country` (`{ id, code, name }`)
- `reviews_count`, `reviews_average`, `updated_at`
