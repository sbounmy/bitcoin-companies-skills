# Bitcoin Companies Skills

Bitcoin treasury research terminal for Claude Code. Search companies, compare holdings, explore on-chain addresses and transactions, read reviews, and generate reports.

Powered by the [Bitcoin Companies API](https://bitcoincompanies.co/api-docs) (free, no auth required).

## Install

```bash
claude plugin add sbounmy/bitcoin-companies-skills
```

## Usage

One command for everything. Just type `/btc` followed by what you want:

### Lookup & Search

| Command | What it does |
|---------|-------------|
| `/btc strategy.com` | Company detail card |
| `/btc Marathon` | Search by name |

### Leaderboard & Rankings

| Command | What it does |
|---------|-------------|
| `/btc top 10 mining` | Top 10 miners |
| `/btc usa` | US companies leaderboard |
| `/btc whales` | Whale-tier companies |
| `/btc countries` | Countries ranked by BTC |
| `/btc tiers` | Tier definitions |

### Compare & Analyze

| Command | What it does |
|---------|-------------|
| `/btc vs strategy.com marathon` | Side-by-side comparison |
| `/btc report etf` | ETF treasury analysis |
| `/btc flow mining` | Net flow & market weather |
| `/btc map` | ASCII world map |

### On-Chain Data

| Command | What it does |
|---------|-------------|
| `/btc addresses strategy.com` | Verified wallet addresses |
| `/btc tx strategy.com` | Recent transactions |
| `/btc proof strategy.com` | Proof of reserves summary |

### Trust & Reviews

| Command | What it does |
|---------|-------------|
| `/btc trust binance` | Trust score & reviews |
| `/btc reviews coinbase.com` | Community reviews |

## Examples

### Look up a company

```
/btc strategy.com
```

```
Strategy (MSTR) — 👑 Sovereign
738,731 BTC (~$52.4B USD) | Verified: 59%
Rank #3 | United States
Supply: 3.52% of 21M
View on site: https://bitcoincompanies.co/strategy.com
```

### Compare companies

```
/btc vs strategy.com marathon
```

Side-by-side table comparing BTC holdings, tier, verification status, rank, and supply percentage.

### Explore on-chain data

```
/btc addresses strategy.com
/btc tx strategy.com --min-btc 100
/btc proof strategy.com
```

Browse verified wallet addresses, filter transactions by size, and view proof of reserves with on-chain verification details.

### Check trust

```
/btc trust binance
```

Shows average review rating, review count, verified BTC percentage, and recent reviews.

### ASCII World Map

```
/btc map
```

Renders a terminal-friendly world map with top 3 countries by BTC holdings marked, plus a top 10 leaderboard.

## API

These skills use the public [Bitcoin Companies API](https://bitcoincompanies.co/api-docs):

- Base URL: `https://bitcoincompanies.co/api/v1`
- Auth: None required
- Rate limit: 60 requests/min per IP
- Format: JSON (Stripe-style responses)

## License

MIT
