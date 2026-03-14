# Bitcoin Companies Skills

Claude Code plugin to query Bitcoin company treasury data. Search companies, browse the leaderboard, and generate treasury reports, all from your terminal.

Powered by the [Bitcoin Companies API](https://bitcoincompanies.co/api-docs) (free, no auth required).

## Install

```bash
claude plugin add sbounmy/bitcoin-companies-skills
```

Or manually copy the `skills/` folder to `~/.claude/skills/`.

## Commands

| Command | Description | Example |
|---------|-------------|---------|
| `/bitcoin-search` | Look up a company by domain or name | `/bitcoin-search strategy.com` |
| `/bitcoin-list` | Display the treasury leaderboard | `/bitcoin-list top 10 mining` |
| `/bitcoin-report` | Generate a treasury analysis report | `/bitcoin-report etf` |

## Usage

### Search a company

```
/bitcoin-search strategy.com
```

```
Strategy (MSTR) — 👑 Sovereign
738,731 BTC (~$52.4B USD) | Verified: 59%
Rank #3 | United States
Supply: 3.52% of 21M
Profile: https://bitcoincompanies.co/strategy.com
```

### Browse the leaderboard

```
/bitcoin-list top 10
```

Returns a formatted table with rank, company, ticker, BTC holdings, tier, verification status, and country.

Filter by category, country, or tier:

```
/bitcoin-list mining
/bitcoin-list usa
/bitcoin-list whales
/bitcoin-list etf verified
```

### Generate a report

```
/bitcoin-report
```

Returns aggregate stats: total companies, total BTC held, breakdowns by category and tier, plus a top 10 table.

Filter by category or country:

```
/bitcoin-report mining
/bitcoin-report united-states
```

## API

These skills use the public [Bitcoin Companies API](https://bitcoincompanies.co/api-docs):

- Base URL: `https://bitcoincompanies.co/api/v1`
- Auth: None required
- Rate limit: 60 requests/min per IP
- Format: JSON (Stripe-style responses)

## License

MIT
