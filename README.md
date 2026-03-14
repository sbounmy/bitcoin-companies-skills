# Bitcoin Companies Skills

Claude Code plugin to query Bitcoin company treasury data. Search companies, browse the leaderboard, and generate treasury reports, all from your terminal.

Powered by the [Bitcoin Companies API](https://bitcoincompanies.co/api-docs) (free, no auth required).

## Install

```bash
claude plugin add sbounmy/bitcoin-companies-skills
```

Or manually copy the `skills/` folder to `~/.claude/skills/`.

## Usage

### `/bitcoin` (recommended)

One command for everything. Just type what you want in plain language:

```
/bitcoin strategy.com           → company detail card
/bitcoin Marathon               → search by name
/bitcoin usa                    → US companies leaderboard
/bitcoin top 10 mining          → top 10 miners
/bitcoin whales                 → whale-tier companies
/bitcoin report etf             → ETF treasury analysis
/bitcoin countries              → countries ranked by BTC
/bitcoin tiers                  → tier definitions
```

### Specific commands

You can also use the targeted commands directly:

| Command | Description | Example |
|---------|-------------|---------|
| `/bitcoin-search` | Look up a company by domain or name | `/bitcoin-search strategy.com` |
| `/bitcoin-list` | Display the treasury leaderboard | `/bitcoin-list top 10 mining` |
| `/bitcoin-report` | Generate a treasury analysis report | `/bitcoin-report etf` |

## Examples

### Look up a company

```
/bitcoin strategy.com
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
/bitcoin top 10
/bitcoin mining
/bitcoin usa verified
```

Returns a formatted table with rank, company, ticker, BTC holdings, tier, verification status, and country.

### Generate a report

```
/bitcoin report
/bitcoin report mining
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
