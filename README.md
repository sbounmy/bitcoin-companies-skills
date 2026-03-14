# Bitcoin Companies Skills

Claude Code plugin to query Bitcoin company treasury data. Search companies, browse the leaderboard, and generate treasury reports, all from your terminal.

Powered by the [Bitcoin Companies API](https://bitcoincompanies.co/api-docs) (free, no auth required).

## Install

```bash
claude plugin add sbounmy/bitcoin-companies-skills
```

## Usage

One command for everything. Just type `/btc` followed by what you want:

```
/btc strategy.com           → company detail card
/btc Marathon               → search by name
/btc usa                    → US companies leaderboard
/btc top 10 mining          → top 10 miners
/btc whales                 → whale-tier companies
/btc report etf             → ETF treasury analysis
/btc countries              → countries ranked by BTC
/btc tiers                  → tier definitions
/btc weather                → global market weather (beta)
/btc map                    → ASCII world map of BTC by country
```

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

### Browse the leaderboard

```
/btc top 10
/btc mining
/btc usa verified
```

Returns a formatted table with rank, company, ticker, BTC holdings, tier, verification status, and country.

### Generate a report

```
/btc report
/btc report mining
```

Returns aggregate stats: total companies, total BTC held, breakdowns by category and tier, plus a top 10 table.

## API

These skills use the public [Bitcoin Companies API](https://bitcoincompanies.co/api-docs):

- Base URL: `https://bitcoincompanies.co/api/v1`
- Auth: None required
- Rate limit: 60 requests/min per IP
- Format: JSON (Stripe-style responses)

## License

MIT
