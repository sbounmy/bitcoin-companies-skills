---
name: bitcoin-search
description: Search and lookup Bitcoin company treasury data. Use when the user asks about a specific company's BTC holdings, tier, or wants to find companies by name.
---

# Bitcoin Search

Look up company Bitcoin treasury data from the Bitcoin Companies API.

## How to use

The user provides a **domain** (e.g., `strategy.com`) or a **search query** (e.g., `Marathon`).

### Step 1: Fetch the data

**If the argument looks like a domain** (contains a dot):
```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}
```

**If the argument is a name or keyword**:
```
WebFetch https://bitcoincompanies.co/api/v1/companies?q={query}&limit=5
```

### Step 2: Format the result

Display a clean summary card:

```
**{name}** ({ticker}) — {tier.emoji} {tier.name}
{btc} BTC (${btc * price} USD) | Verified: {verified_percentage}%
Rank #{rank} | {country.name}
Supply: {supply_percentage}% of 21M
Profile: https://bitcoincompanies.co/{domain}
```

If multiple results (search), show a compact table instead.

### Step 3 (optional): Show weather

If the user asked about activity or sentiment, also fetch:
```
WebFetch https://bitcoincompanies.co/api/v1/companies/{domain}/weather
```

## API Reference

```
Base: https://bitcoincompanies.co/api/v1

GET /companies?q={query}&limit=5     Search by name, ticker, or domain
GET /companies/{domain}              Full company detail
GET /companies/{domain}/weather      Company market weather
GET /price                           Current BTC price (for USD conversion)
```

## Response format

Company detail returns:
- `id` (domain), `name`, `ticker`, `category`
- `rank` (integer position), `tier` (`{ id, object, name, emoji }`)
- `treasury` (`{ btc, verified_btc, claimed_btc, verified_percentage, supply_percentage }`)
- `country` (`{ id, code, name }`)
- `reviews` (`{ count, average_rating }`)
- `profile_url`, `updated_at`
