---
name: btc
description: "Answer ANY question about Bitcoin companies, treasuries, holdings, rankings, reviews, or trust. Works with both commands AND natural language. Use this skill whenever the user asks about: Bitcoin company data, BTC holdings, treasury rankings, company comparisons, on-chain addresses/transactions, proof of reserves, reviews/trust/reputation, Bitcoin leaderboards, mining/ETF/exchange companies, or country-level BTC stats. Examples: 'strategy.com', 'how much bitcoin does coinbase hold?', 'what are the top mining companies?', 'can I trust binance?', 'compare marathon vs riot', 'show me strategy bitcoin purchases', 'which countries hold the most bitcoin?', 'is kraken safe?', 'what tier is tesla?'"
---

# Bitcoin Companies

Answer any question about Bitcoin companies. Works with both structured commands and natural language.

## Performance Rules

**CRITICAL: Always fetch in parallel when possible.** WebFetch calls are slow (~15s each). Never fetch sequentially when calls are independent.

- If you need to search by name first (`/companies?q=X`), do that FIRST, then fetch all remaining endpoints IN PARALLEL in a single tool call block.
- If the domain is already known, fetch ALL endpoints in parallel immediately.
- Example: REVIEWS intent with domain known → fetch `/companies/{domain}` AND `/companies/{domain}/reviews` in the SAME tool call block.

## Step 1: Route the query

Classify the user's input into an intent. The input can be a structured command OR a natural language question. Extract the key entity (company name/domain) and action from the query.

| Intent | Triggers (commands OR natural language) | Examples |
|--------|----------------------------------------|----------|
| **LOOKUP** | Contains a domain (has a `.`), asks about a specific company, "how much BTC does X hold?", "tell me about X", "what tier is X?" | `strategy.com`, `"how much bitcoin does MicroStrategy hold?"`, `"what tier is Tesla?"` |
| **REPORT** | Asks for aggregate data, summary, overview, stats, "how much total BTC?", starts with "report" | `report mining`, `"what's the total BTC held by all companies?"`, `"give me an overview of ETF holdings"` |
| **LIST** (category) | Mentions a category (mining, etf, exchange, stocks, custody, software, private), "show me X companies" | `mining`, `"what are the top mining companies?"`, `"list all ETF companies"` |
| **LIST** (country) | Mentions a country name/code, "companies in X", "which companies are in X?" | `usa`, `france`, `"which bitcoin companies are in Japan?"` |
| **LIST** (region) | Mentions a region/continent, "companies in Europe", "Asian bitcoin companies" | `europe`, `"top bitcoin companies in Asia"`, `"North American holders"` |
| **LIST** (tier) | Mentions a tier name (whale, shark, sovereign, shrimp, dolphin, etc.), "which companies are whales?" | `whales`, `"show me all sovereign-tier companies"` |
| **LIST** | Contains "top N", "list", "rank", "leaderboard", "biggest", "largest" | `top 10`, `"who are the biggest bitcoin holders?"`, `"show the leaderboard"` |
| **LIST** (special) | Asks about countries or tiers as categories | `"list countries"`, `"what are the tiers?"`, `"show tier definitions"` |
| **MAP** | Contains "map", "world", "globe", "where", asks about geographic distribution | `map`, `"where are bitcoin companies located?"`, `"show me a world map"` |
| **HISTORY** | Asks about purchase/acquisition history, timeline, "when did X buy bitcoin?", starts with "history"/"claims" | `history strategy.com`, `"when did MicroStrategy start buying bitcoin?"`, `"show me coinbase acquisitions"` |
| **VS** | Compares two companies, "X vs Y", "compare X and Y", "how does X compare to Y?" | `vs strategy.com metaplanet.com`, `"compare Marathon and Riot"`, `"who holds more, Coinbase or Kraken?"` |
| **FLOW** | Asks about inflows/outflows, buying/selling activity, net flow, starts with "flow", mentions "weather"/"sentiment" | `flow mining`, `"who's been buying lately?"`, `"is anyone selling bitcoin?"`, `"market sentiment"` |
| **ADDRESSES** | Asks about wallet addresses, tracked addresses, starts with "addresses"/"addrs" | `addresses strategy.com`, `"what wallets does Coinbase use?"`, `"show tracked addresses for Marathon"` |
| **TX** | Asks about recent transactions, on-chain activity, starts with "tx"/"transactions" | `tx strategy.com`, `"what are the latest transactions for Strategy?"`, `"show me large transfers from Coinbase"` |
| **PROOF** | Asks about proof of reserves, verification, cryptographic proof, starts with "proof"/"por" | `proof strategy.com`, `"does Coinbase have proof of reserves?"`, `"how is Marathon verified?"` |
| **REVIEWS** | Asks about reviews, trust, safety, reputation, reliability, "can I trust X?", "is X safe?", "what do people think of X?" | `trust binance`, `"what are binance reviews?"`, `"is Kraken reliable?"`, `"what do people say about Coinbase?"` |
| **WRITE_REVIEW** | Wants to write/post/leave/submit a review for a company | `"leave a review for binance"`, `"write a review for strategy.com"`, `"post a review"`, `"review coinbase 5 stars"` |
| **LOOKUP** (fallback) | Everything else that mentions a company name | `Marathon`, `Coinbase`, `"tell me about MicroStrategy"` |

**When extracting from natural language:**
- Find the company name/domain in the question
- If user mentions a company by name (not domain), search via `/companies?q={name}&limit=1` first
- Map "how much bitcoin", "holdings", "treasury" → LOOKUP
- Map "buy", "purchase", "acquire", "when did they" → HISTORY
- Map "wallet", "address" → ADDRESSES
- Map "trust", "safe", "legit", "reliable", "reputation", "reviews", "what do people think" → REVIEWS
- Map "compare", "vs", "versus", "better", "who holds more" → VS

**Country aliases to resolve:**

| Alias | Resolves to |
|-------|-------------|
| `usa`, `us` | `united-states` |
| `uk` | `united-kingdom` |

---

## LOOKUP intent

Look up a specific company by domain or search by name.

### Fetch

**If input contains a dot** (domain) — fetch BOTH in parallel:
```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
WebFetch https://bitcoincompanies.co/api/v1/price
```

**Otherwise** (name search) — fetch search + price in parallel:
```
WebFetch https://bitcoincompanies.co/api/v1/companies?q={query}&limit=5
WebFetch https://bitcoincompanies.co/api/v1/price
```

### Format

**Single company** (domain lookup or single search result):
```
**{name}** ({ticker}) — {tier.emoji} {tier.name}
{btc} BTC (${btc * price} USD) | Verified: {verified_percentage}%
Rank #{rank} | {country.name} | {average_rating}★ ({count} reviews)
Supply: {supply_percentage}% of 21M
View on site: {app_url}
```

If `reviews.count` is 0, show "No reviews yet" instead of the rating.

**Multiple results** (name search):

| # | Company | Ticker | BTC | Tier | Rating | Country |
|---|---------|--------|-----|------|--------|---------|
| 1 | ... | ... | ... | ... | 4.2★ (15) | ... |

Use `reviews_average` and `reviews_count` from list items. Show "—" if no reviews.

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
| `europe` / `asia` / `north america` | `region={slug}` (e.g., `region=western-europe`) |
| Any region name | `region={slug}` |
| `whale` / `whales` | `tier=whale` |
| `sovereign` | `tier=sovereign` |
| Any tier name | `tier={tier}` |
| `verified` / `verified only` | `only_verified=true` |
| `best reviewed` / `highest rated` | `sort=best_reviewed` |
| `page 2` | `page=2` |

Default: `limit=25` with no filters.

**Region slugs:** `north-america`, `south-america`, `western-europe`, `eastern-europe`, `asia`, `middle-east`, `africa`, `oceania`

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

| # | Company | Ticker | BTC | Tier | Rating | Verified | Country |
|---|---------|--------|-----|------|--------|----------|---------|
| 1 | Strategy | MSTR | 500,000 | 👑 Sovereign | 4.5★ (23) | 100% | US |
| ... |

Use `reviews_average` and `reviews_count`. Show "—" if no reviews.

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

## HISTORY intent

Show a company's Bitcoin acquisition timeline.

### Fetch

```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/claims?limit=10&page={page}
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
```

If user provides a name instead of domain, search first:
```
WebFetch https://bitcoincompanies.co/api/v1/companies?q={query}&limit=1
```
Then use the `id` (domain) from the first result.

### Format

```
## {name} — Acquisition History

| Date | BTC | Type | Source | Notes |
|------|-----|------|--------|-------|
| 2024-11-05 | +15,400 | SEC Filing | [link] | Largest single purchase |
| 2024-10-15 | +7,420 | SEC Filing | [link] | |
| 2024-09-01 | -500 | Sale | [link] | Partial divestiture |
| ... |

Total: {total_count} events since {earliest_date}
View on site: {app_url}
```

Format `btc` with `+` prefix for purchases, `-` for sales. Use comma separators.
Use `claimed_at` for the Date column. Use `source_type` for Type (capitalize: "sec_filing" → "SEC Filing").
If `source_url` exists, make Type a markdown link.
Show last 10 by default. If `has_more`, append: "Page {page}. Say 'page 2' for more."

---

## VS intent

Side-by-side comparison of two companies.

### Parse

Extract two company identifiers from the query. They can be domains or names.
- `vs strategy.com metaplanet.com` → domain1=strategy.com, domain2=metaplanet.com
- `compare Marathon Riot` → search for each, use top result's domain
- `vs strategy.com Metaplanet` → mix of domain and name

If both identifiers are identical, respond: "Cannot compare a company with itself."

### Fetch (MAXIMIZE parallel calls)

**If both are domains** — fetch ALL 3 in parallel:
```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain1}
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain2}
WebFetch https://bitcoincompanies.co/api/v1/price
```

**If names need search** — search both + price in parallel first, then fetch details in parallel:
```
# Step 1 (parallel):
WebFetch https://bitcoincompanies.co/api/v1/companies?q={name1}&limit=1
WebFetch https://bitcoincompanies.co/api/v1/companies?q={name2}&limit=1
WebFetch https://bitcoincompanies.co/api/v1/price
# Step 2 (parallel, using domains from step 1):
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain1}
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain2}
```

If either company returns 404, show: "Company not found: {identifier}"

### Format

```
## {name1} vs {name2}

|                  | {name1}          | {name2}          |
|------------------|------------------|------------------|
| Tier             | {tier.emoji} {tier.name} | {tier.emoji} {tier.name} |
| BTC              | {btc}            | {btc}            |
| USD Value        | ${btc * price}   | ${btc * price}   |
| Rank             | #{rank}          | #{rank}          |
| Verified         | {verified_pct}%  | {verified_pct}%  |
| Supply %         | {supply_pct}%    | {supply_pct}%    |
| Category         | {category}       | {category}       |
| Country          | {country.name}   | {country.name}   |
| Reviews          | {avg}★ ({count}) | {avg}★ ({count}) |

View: {app_url1} | {app_url2}
```

Format BTC and USD with comma separators. Use `app_url` from each company response.

---

## FLOW intent

Net inflow/outflow and sentiment. Aliases: "flow", "weather", "sentiment".

### Parse arguments

The command can combine a target + time window:

| Argument | Detected by | Maps to |
|----------|-------------|---------|
| Domain | Contains a dot | Company flow: `/companies/{domain}/weather` |
| Category | mining, etf, exchange, etc. | `?category={cat}` on global weather |
| Country | usa, japan, france, etc. | `?location={slug}` on global weather |
| "yesterday" or "d1" | Keyword match | `?window=d1` |
| "monthly" or "this month" | Keyword match | `?window=m0&scale=monthly` |
| "yearly" or year (2025) | Keyword or 4-digit number | `?window=y{year}&scale=yearly` |

Default: global flow, today (`d0`).

Examples:
- `/btc flow` → `GET /weather?window=d0`
- `/btc flow strategy.com` → `GET /companies/strategy.com/weather?window=d0`
- `/btc flow mining monthly` → `GET /weather?category=mining&window=m0&scale=monthly`
- `/btc flow usa yesterday` → `GET /weather?location=united-states&window=d1`
- `/btc flow strategy.com 2025` → `GET /companies/strategy.com/weather?window=y2025&scale=yearly`

### Fetch

**Global flow** (no domain):
```
WebFetch https://bitcoincompanies.co/api/v1/weather?{params}
```

**Company flow** (domain provided):
```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/weather?{params}
```

### Format (global)

```
## Bitcoin Flow — {period_label}

{condition.emoji} Score: {score}/100 — {condition.label}
Net Flow: {net_flow > 0 ? "+" : ""}{net_flow} BTC

Buying: {buying_companies} companies | Selling: {selling_companies} companies

TOP BUYERS                          TOP SELLERS
{name1}         +{btc} BTC          {name1}         -{btc} BTC
{name2}         +{btc} BTC          {name2}         -{btc} BTC
{name3}         +{btc} BTC

View on site: {app_url}
```

### Format (company)

```
## {company_name} — Flow ({period_label})

{condition.emoji} Score: {score}/100 — {condition.label}
Net Flow: {net_flow > 0 ? "+" : ""}{net_flow} BTC

Receiving: {receiving_addresses} addresses | Sending: {sending_addresses} addresses

TOP RECEIVERS                       TOP SENDERS
{label1}        +{btc} BTC         {label1}        -{btc} BTC
{label2}        +{btc} BTC

View on site: {app_url}
```

If score is null (no data), show: "{condition.emoji} No Signal — insufficient on-chain data for this period."

**Beta**: Always append: "Flow data is in beta. Historical on-chain coverage may be incomplete for some companies."

---

## ADDRESSES intent

List tracked Bitcoin addresses for a company with balances.

### Parse

- `addresses strategy.com` → domain, no filter
- `addresses strategy.com verified` or `addrs strategy.com verified` → domain, state=verified

If input is a name (no dot), search first via `/companies?q={name}&limit=1`.

### Fetch

```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/addresses?sort=balance&limit=25&page={page}
```

If "verified" keyword present, add `&state=verified`.

### Format

```
## {company_name} — Tracked Addresses ({total_count} total)

| # | Address          | Label           | Balance    | State    | Explorer |
|---|------------------|-----------------|------------|----------|----------|
| 1 | bc1q...abc       | Cold Storage #1 | 5,000 BTC  | verified | [Arkham](explorer_url) |
| 2 | bc1q...def       | Treasury        | 3,200 BTC  | verified | [Arkham](explorer_url) |
| 3 | 3J98t...ghi      | -               | 1,800 BTC  | pending  | [Arkham](explorer_url) |
| ... |

Page {page}. {has_more ? "Say 'page 2' for more." : ""}
View on site: {app_url}
```

Truncate addresses to first 6 + last 4 characters. Format BTC with comma separators.
Use `explorer_url` from the API response for the Arkham link.

---

## TX intent

Recent on-chain transactions for a company.

### Parse

- `tx strategy.com` → domain, default filters
- `tx strategy.com --large` → domain, min_btc=1
- `transactions strategy.com in` → domain, direction=in
- `tx strategy.com out` → domain, direction=out

If input is a name (no dot), search first via `/companies?q={name}&limit=1`.

### Fetch

```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/transactions?sort=recent&limit=25&page={page}
```

Add filters:
- `--large` keyword → `&min_btc=1`
- `in` keyword → `&direction=in`
- `out` keyword → `&direction=out`

### Format

```
## {company_name} — Recent Transactions ({total_count} total{total_count_capped ? "+" : ""})

| Date       | Dir | Amount     | Address          | Label           | Explorer |
|------------|-----|------------|------------------|-----------------|----------|
| 2026-03-14 | IN  | +150.0 BTC | bc1q...abc       | Cold Storage #1 | [Arkham](explorer_url) |
| 2026-03-13 | OUT | -25.0 BTC  | bc1q...def       | Treasury        | [Arkham](explorer_url) |
| ... |

Page {page}. {has_more ? "Say 'page 2' for more." : ""}
```

Format direction as uppercase IN/OUT. Prefix amount with + for IN, - for OUT.
Truncate addresses to first 6 + last 4 characters.
If `total_count_capped` is true, show "10,000+" instead of exact count.
Use `explorer_url` from the API response for the Arkham link.

---

## PROOF intent

Proof of Reserve verification data. Shows only verified addresses with cryptographic signatures.

### Fetch

```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/addresses?state=verified&limit=10&page={page}
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
```

If input is a name (no dot), search first via `/companies?q={name}&limit=1`.

### Format

```
## {company_name} — Proof of Reserve

Verified: {verified_btc} BTC / {claimed_btc} BTC claimed ({verified_percentage}%)

### Verified Addresses ({total_count})

1. bc1q...abc (Cold Storage #1) — 5,000 BTC
   Message: "Bitcoin Companies verification for strategy.com"
   Signature: H3kJ9...xYz
   Explorer: [Arkham](explorer_url)

2. bc1q...def (Treasury) — 3,200 BTC
   Message: "Bitcoin Companies verification for strategy.com"
   Signature: G7mN2...aBc
   Explorer: [Arkham](explorer_url)

... (showing {limit} of {total_count}{has_more ? ", say 'page 2' for more" : ""})

View on site: {app_url}
```

Use `verification_message` and `signature` from the API response (only included when state=verified).
Truncate signatures to first 5 + last 3 characters.
Use `explorer_url` from the API response for the Arkham link.

---

## REVIEWS intent

Community reviews and trust signal for a company. Also triggered by trust queries like "can I trust X?", "is X safe?".

### Parse

- `reviews binance.com` → domain, list reviews
- `reviews coinbase 5` → search name, filter rating=5
- `trust binance` → search name, trust summary + reviews
- `can I trust coinbase` → search name, trust summary + reviews
- `is binance safe` → search name, trust summary + reviews

If input is a name (no dot), search first via `/companies?q={name}&limit=1`, then use the `id` (domain) from the result.

Trust queries (contains "trust", "safe", "reliable", "legit") trigger a trust summary before the reviews.

### Fetch

**IMPORTANT: Fetch company detail AND reviews in PARALLEL** (both only need the domain):

```
# If name search was needed, do it first, then parallel:
WebFetch https://bitcoincompanies.co/api/v1/companies?q={name}&limit=1
# Then IN PARALLEL (same tool call block):
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/reviews?sort=recent&limit=10&page={page}

# If domain is already known, fetch BOTH in parallel immediately:
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/reviews?sort=recent&limit=10&page={page}
```

If rating filter provided, add `&rating={n}`.

**Language filtering:** Detect the user's language from the conversation (e.g., French user → `&language=fr`). Always add `&language={code}` to show reviews in the user's language. If no reviews match, retry without the language filter and note that reviews are in mixed languages.

### Format

**If trust query, prepend trust summary:**
```
## Can you trust {name}?

Rating: {reviews.average_rating}/5 ({reviews.count} reviews) | Verified: {treasury.verified_percentage}% on-chain
Rank #{rank} | {tier.emoji} {tier.name}

---
```

**Reviews list:**
```
## {name} — Reviews ({total_count} total)

**** (4/5) — "Solid exchange but high fees"
By CryptoTrader42 via Trustpilot — Feb 15, 2026
Been using Binance for 3 years...

  > Reply from Binance Team: Thank you for your feedback...

** (2/5) — "Withdrawal issues"
By Anonymous via Bitcoin Companies — Jan 20, 2026
Tried to withdraw 0.5 BTC and it took 3 days...

... (showing {limit} of {total_count}{has_more ? ", say 'page 2' for more" : ""})

View on site: {app_url}
```

Render stars as `*` repeated per rating (e.g., `***` for 3/5).
If `reply_text` exists, show as indented blockquote with `reply_author`.
Truncate `body` to first 200 characters if longer, append "..."
Format `published_at` as "Mon DD, YYYY".

---

## WRITE_REVIEW intent

Post a review for a company via L402 Lightning payment. The agent collects review data, submits it, and displays a QR code for the human to pay.

### Parse

Extract from the query:
- Company name or domain
- Rating (1-5) if mentioned ("5 stars", "4/5", "rate it 3")

If input is a name (no dot), search first via `/companies?q={name}&limit=1`.

### Flow

**Step 1: Fetch company info to know the category**

```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
```

**Step 2: Ask for criteria ratings based on company category**

Present relevant criteria as a quick checklist. The user rates each 1-5 (or skips). The agent uses these to compute the overall rating and generate the review body.

| Category | Criteria |
|----------|----------|
| **exchange** | Security, Fees, Ease of use, Customer support, Withdrawal speed |
| **mining** | Transparency, Hash rate reliability, Environmental practices, Financial reporting |
| **etf** | Expense ratio, Tracking accuracy, Liquidity, Accessibility |
| **custody** | Security, Insurance coverage, Audit frequency, Multi-sig support |
| **software** | Product quality, Bitcoin integration, Transparency, Value for money |
| **stocks** | Treasury strategy, Shareholder communication, Transparency, Execution |
| **private** | Transparency, Proof of reserves, Community trust, Track record |

Ask in ONE message with star scale:
```
Rate Binance (exchange) — type 5 numbers (1-5):

Security:      ☆☆☆☆☆
Fees:          ☆☆☆☆☆
Ease of use:   ☆☆☆☆☆
Support:       ☆☆☆☆☆
Withdrawals:   ☆☆☆☆☆

e.g. "4 3 5 4 3" or skip any with 0
```

**Step 3: Confirm with filled stars, then generate review**

After user responds (e.g. `4 3 5 4 3`), show confirmation:
```
Security:      ★★★★☆ (4/5)
Fees:          ★★★☆☆ (3/5)
Ease of use:   ★★★★★ (5/5)
Support:       ★★★★☆ (4/5)
Withdrawals:   ★★★☆☆ (3/5)

Overall: ★★★★☆ (4/5)
```

Then ask the user to write their review:
```
Write your review (2-3 sentences):
```
- `overall_rating` = average of provided criteria ratings (rounded)
- `body` = the user's own words. Do NOT generate or write the review body for them. Wait for their input.
- Do NOT include a `title` (unnecessary)

Use `★` (U+2605) for filled stars and `☆` (U+2606) for empty stars.

**Step 2: Submit review → get 402 with invoice**

```bash
curl -s -X POST https://bitcoincompanies.co/api/v1/companies/{domain}/reviews \
  -H "Content-Type: application/json" \
  -d '{"overall_rating": 4, "body": "Solid security and easy to use, but fees are on the high side. Withdrawals process within a few hours."}'
```

The API returns 402 with a Lightning invoice and a QR code:

```json
{
  "error": {
    "type": "payment_required",
    "message": "Pay 100 sats to publish this review",
    "invoice": "lnbc100n1p...",
    "amount_sats": 100,
    "payment_hash": "abc123...",
    "qr": "██████████████  ████...\n██          ██  ..."
  }
}
```

**Step 3: Display QR code to the human**

Show the QR code from the `qr` field so the human can scan and pay with any Lightning wallet:

```
Your review is ready! Please scan this QR code with your Lightning wallet to pay 100 sats:

██████████████  ████████  ██    ██  ██████████████
██          ██        ████  ██      ██          ██
██  ██████  ██    ██  ██  ████  ██  ██  ██████  ██
...

Or copy this invoice into your wallet:
lnbc100n1p...
```

**Step 4: Confirm publication**

After the human pays, the review is auto-published via webhook. Verify by fetching the reviews list:

```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/reviews?sort=recent&limit=1
```

Then confirm: "Your review has been published!"

If the review isn't published yet (payment still processing), tell the human to wait a moment and check again.

### Format

**After collecting criteria, show confirmation + submit:**
```
## Review for {company_name}

★★★★☆ (4/5)
Solid security and easy to use, but fees are on the high side.
Withdrawals process within a few hours.

Submitting...
```

**After 402 response, show QR:**
```
Review submitted! Pay 100 sats to publish:

{qr field content, displayed as-is}

Or copy invoice: {invoice}

After payment, your review will be published automatically.
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
  ?region=western-europe              Filter by region slug
  ?sort=rank|verified|best_reviewed   Sort order
  ?only_verified=true                 Verified companies only
  ?limit=25&page=1                    Pagination (max 100)

GET /companies/{domain}               Company detail (by domain)
GET /companies/{domain}/weather       Company market weather / flow
GET /companies/{domain}/claims        Company claims history
GET /companies/{domain}/addresses     Company tracked addresses
  ?state=verified                     Only verified addresses
  ?sort=balance|recent                Sort order (default: balance)
GET /companies/{domain}/transactions  Company on-chain transactions
  ?min_btc=0.01                       Minimum BTC amount (default: 0.01)
  ?direction=in|out                   Filter by direction
  ?sort=recent|amount                 Sort order (default: recent)
GET /companies/{domain}/reviews       Company reviews
  ?rating=1-5                         Filter by star rating
  ?language=en                        Filter by language code (en, fr, de, etc.)
  ?sort=recent|rating                 Sort order (default: recent)
POST /companies/{domain}/reviews      Create review (L402 payment required)
  Body: { overall_rating, title?, body, language? }
  → 402 with Lightning invoice + QR code
  → Pay invoice, then review auto-publishes via webhook

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

**Address:**
- `id` (address string), `object` ("address"), `address`, `label`, `state`
- `balance_btc`, `explorer_url` (Arkham link)
- When `state=verified`: `verification_message`, `signature`

**Transaction:**
- `id` (txid), `object` ("transaction"), `txid`, `address`, `address_label`
- `amount_btc` (absolute value), `direction` ("in"/"out"), `confirmed`
- `block_height`, `block_time`, `explorer_url` (Arkham link)

**Review:**
- `id`, `object` ("review"), `state` ("pending"/"published"/"hidden")
- `overall_rating` (1-5), `title`, `body`
- `author_name`, `source`, `language` (ISO code: "en", "fr", "de", etc.), `published_at`
- `reply_text`, `reply_author` (nullable, for company replies)

**All responses include `app_url`** — always use this field for "View on site" links. Never hardcode URLs.
