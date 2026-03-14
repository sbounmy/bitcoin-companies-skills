---
name: btc
description: "Ask anything about Bitcoin companies. Examples: 'strategy.com', 'top 10 mining', 'usa', 'report etf', 'whales', 'Marathon', 'countries', 'map'"
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
| Contains "weather", "sentiment", "activity" | `weather for strategy.com`, `market sentiment` | **LOOKUP** + weather (BETA) |
| Contains "map", "world", "globe" | `map`, `world map`, `globe` | **MAP** |
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

**BETA NOTE**: Always append this disclaimer when showing weather data:
> **Beta**: Weather is in beta. We're still sourcing historical on-chain data for many companies, so coverage may be incomplete.

### Format

**Single company** (domain lookup or single search result):
```
**{name}** ({ticker}) — {tier.emoji} {tier.name}
{btc} BTC (${btc * price} USD) | Verified: {verified_percentage}%
Rank #{rank} | {country.name}
Supply: {supply_percentage}% of 21M
View on site: {app_url}
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

Always append "View on site: {app_url}" using the `app_url` field from the API response.

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

View on site: {app_url}
```

---

## MAP intent

Draw a pure ASCII world map showing BTC holdings by country.

### Fetch

```
WebFetch https://bitcoincompanies.co/api/v1/countries?limit=50
```

### Format

**IMPORTANT: No emoji.** Emoji have inconsistent widths and break VS Code / terminal alignment.

The map is 69 columns x 23 rows. Place `[1]` `[2]` `[3]` markers for the top 3 countries by replacing the character at the given (col, row) position. Row 0 is the first line of the map, col 0 is the first character.

**Pre-computed marker positions** (Mercator projection from capital city lat/lon):

| Code | Col | Row | Country |
|------|-----|-----|---------|
| ar | 23 | 15 | Argentina |
| au | 63 | 15 | Australia |
| at | 37 | 5 | Austria |
| bs | 19 | 8 | Bahamas |
| bm | 22 | 7 | Bermuda |
| bt | 51 | 8 | Bhutan |
| br | 25 | 13 | Brazil |
| vg | 22 | 9 | British Virgin Islands |
| ca | 19 | 5 | Canada |
| ky | 18 | 9 | Cayman Islands |
| cn | 56 | 6 | China |
| cr | 18 | 10 | Costa Rica |
| cw | 21 | 10 | Curacao |
| sv | 17 | 9 | El Salvador |
| fi | 39 | 2 | Finland |
| fr | 34 | 5 | France |
| de | 37 | 4 | Germany |
| gi | 33 | 7 | Gibraltar |
| hk | 56 | 8 | Hong Kong |
| in | 49 | 8 | India |
| id | 54 | 12 | Indonesia |
| il | 41 | 7 | Israel |
| jp | 61 | 7 | Japan |
| je | 34 | 5 | Jersey |
| lt | 39 | 4 | Lithuania |
| lu | 35 | 4 | Luxembourg |
| mt | 37 | 7 | Malta |
| mx | 15 | 9 | Mexico |
| nl | 35 | 4 | Netherlands |
| nz | 68 | 16 | New Zealand |
| ng | 35 | 10 | Nigeria |
| kp | 58 | 6 | North Korea |
| no | 36 | 2 | Norway |
| pa | 19 | 10 | Panama |
| ph | 57 | 9 | Philippines |
| pl | 38 | 4 | Poland |
| ru | 41 | 3 | Russia |
| sc | 45 | 12 | Seychelles |
| sg | 54 | 11 | Singapore |
| si | 37 | 5 | Slovenia |
| za | 39 | 14 | South Africa |
| kr | 58 | 6 | South Korea |
| es | 33 | 6 | Spain |
| se | 37 | 3 | Sweden |
| ch | 35 | 5 | Switzerland |
| tw | 57 | 8 | Taiwan |
| th | 53 | 9 | Thailand |
| tr | 40 | 6 | Turkey |
| ae | 44 | 8 | UAE |
| ua | 40 | 4 | Ukraine |
| gb | 34 | 4 | United Kingdom |
| us | 19 | 6 | United States |
| ve | 21 | 10 | Venezuela |

**Base map** (from github.com/HenrySeed/Terminal-World-Map):

```
          . _..::__:  ,-"-"._       |]       ,     _,.__
  _.___ _ _<_>`!(._`.`-.    /        _._     `_ ,_/  '  '-._.---.-..__
.{     " " `-==,',._\{  \  / {)     / _ ">_,-' `                 /-/_
 \_.:--.       `._ )`^-. "'      , [_/(                       __,/-'
'"'     \         "    _L       |-_,--'                )     /. (|
         |           ,'         _)_.\\._<> {}              _,' /  '
         `.         /          [_/_'` `"(                <'}  )
          \\    .-. )          /   `-'"..' `:._          _)  '
   `        \  (  `(          /         `:\  > \  ,-^.  /' '
             `._,   ""        |           \`'   \|   ?_)  {\
                `=.---.       `._._       ,'     "`  |' ,- '.
                  |    `-._        |     /          `:`<_|=--._
                  (        >       .     | ,          `=.__.`-'\
                   `.     /        |     |{|              ,-.,\     .
                    |   ,'          \   / `'            ,"     \
                    |  /             |_'                |  __  /
                    | |                                 '-'  `-'   \.
                    |/                                        "    /
                    \.                                            '

                     ,/           ______._.--._ _..---.---------.
__,-----"-..?----_/ )\    . ,-'"             "                  (__--/
                      /__/\/
```

**Instructions:**
- NO EMOJI anywhere
- Start with the base map above
- Look up the top 3 countries from the API response in the position table
- Replace the character at (col, row) with `[1]`, `[2]`, `[3]` respectively
- If a marker overwrites map characters, that's fine — the marker takes priority
- Below the map, add a legend and top 10 leaderboard:

```
 BITCOIN TREASURY WORLD MAP

 ...map with [1] [2] [3] placed...

 [1] United States    4,810,110 BTC
 [2] Malta              702,478 BTC
 [3] South Korea        239,042 BTC

 TOP 10 BY BTC HOLDINGS
 --------------------------------------------------------------------------
  #1   US  United States     4,810,110 BTC  (221 companies)
  ...
  #10  XX  Country Name          X,XXX BTC  (  X companies)
 --------------------------------------------------------------------------
 View on site: https://bitcoincompanies.co/countries
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
- `app_url` (canonical page URL on bitcoincompanies.co), `updated_at`

**Company list item:**
- `id`, `name`, `ticker`, `category`, `logo_url`
- `rank`, `tier` (`{ id, name, emoji }`)
- `btc`, `verified_percentage`, `supply_percentage`
- `country` (`{ id, code, name }`)
- `app_url` (canonical page URL on bitcoincompanies.co)
- `reviews_count`, `reviews_average`, `updated_at`

**All responses include `app_url`** — always use this field for "View on site" links. Never hardcode URLs.
