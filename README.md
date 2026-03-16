# Bitcoin Companies Skills

Bitcoin treasury data in your terminal. Ask anything about Bitcoin companies, holdings, rankings, reviews, and on-chain data.

Works with **commands** and **plain English**. No API key needed.

https://github.com/sbounmy/bitcoin-companies-skills/releases/download/v2.1.0/demo.mp4

## Install

```bash
claude plugin add sbounmy/bitcoin-companies-skills
```

## Just ask

You don't need to memorize commands. Just ask naturally:

```
how much bitcoin does coinbase hold?
can I trust binance?
what are the top mining companies?
compare marathon vs riot
who are the biggest bitcoin holders in europe?
which companies have the best reviews?
```

Or use `/btc` with structured commands for power users.

## Commands

### Company lookup

| Command | What it does |
|---------|-------------|
| `/btc strategy.com` | Company detail card with BTC, rank, tier |
| `/btc Marathon` | Search by name |
| `how much bitcoin does Tesla hold?` | Natural language lookup |

### Leaderboard & rankings

| Command | What it does |
|---------|-------------|
| `/btc top 10` | Top 10 by BTC holdings |
| `/btc top 10 mining` | Top 10 miners |
| `/btc usa` | US companies |
| `/btc europe` | European companies |
| `/btc whales` | Whale-tier companies |
| `/btc countries` | Countries ranked by BTC |
| `/btc tiers` | Tier definitions |
| `who are the biggest holders in asia?` | Natural language filtering |
| `which companies have the best reviews?` | Sort by review rating |

### Compare & analyze

| Command | What it does |
|---------|-------------|
| `/btc vs strategy.com marathon` | Side-by-side comparison |
| `/btc report etf` | ETF treasury analysis |
| `/btc flow mining` | Net flow and market sentiment |
| `/btc map` | ASCII world map with top holders |
| `compare coinbase and kraken` | Natural language comparison |

### On-chain data

| Command | What it does |
|---------|-------------|
| `/btc addresses strategy.com` | Tracked wallet addresses |
| `/btc tx strategy.com` | Recent transactions |
| `/btc tx strategy.com --large` | Large transactions only |
| `/btc proof strategy.com` | Proof of reserves |
| `show me coinbase wallet addresses` | Natural language |

### Trust & reviews

| Command | What it does |
|---------|-------------|
| `/btc trust binance` | Trust score and reviews |
| `/btc reviews coinbase.com` | Community reviews |
| `is kraken safe?` | Natural language trust query |
| `what do people think of binance?` | Reviews in your language |

### History

| Command | What it does |
|---------|-------------|
| `/btc history strategy.com` | Acquisition timeline |
| `when did MicroStrategy start buying bitcoin?` | Natural language |

## Example output

```
/btc strategy.com

Strategy (MSTR) — 👑 Sovereign
738,731 BTC (~$52.4B USD) | Verified: 59%
Rank #3 | United States
Supply: 3.52% of 21M
View on site: https://bitcoincompanies.co/strategy.com
```

```
/btc map

 BITCOIN TREASURY WORLD MAP

          . _..::__:  ,-"-"._       |]       ,     _,.__
  _.___ _ _<_>`!(._`.`-.    /        _._     `_ ,_/  '  '-._.---.-..__
.{     " " `-==,',._\{  \  / {)     / _ ">_,-' `                 /-/_
 \_.:--.       `._ )`^-. [1]   , [_/(                       __,/-'
'"'     \         "    _L       |-_,--'                )     /. (|
         |           ,'         _)_.\\._<> {}              _,' /  '
         `.         /          [_/_'` `"(                <'}  )
          \\    .-. )          /   `-'"..' `[2]_          _)  '
   `        \  (  `(          /         `:\  > \  ,-^.  /' '
             `._,   ""        |           \`'   \|   ?_)  {\
                `=.---.       `._._       ,'     "`  |' ,- '.

 [1] United States    4,810,110 BTC
 [2] Malta              702,478 BTC
 [3] South Korea        239,042 BTC
```

## API

Powered by the public [Bitcoin Companies API](https://bitcoincompanies.co/api-docs):

- Base: `https://bitcoincompanies.co/api/v1`
- Auth: None required
- Rate limit: 60 req/min per IP
- Format: JSON (Stripe-style with `id` + `object` on every resource)
- Docs: [bitcoincompanies.co/api-docs](https://bitcoincompanies.co/api-docs)

## License

MIT
