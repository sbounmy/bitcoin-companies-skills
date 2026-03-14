# Bitcoin Companies Skills

Claude Code plugin to query Bitcoin company treasury data. Search companies, browse the leaderboard, and generate treasury reports, all from your terminal.

Powered by the [Bitcoin Companies API](https://bitcoincompanies.co/api-docs) (free, no auth required).

## Install

```bash
claude plugin add sbounmy/bitcoin-companies-skills
```

## Usage

One command for everything. Just type `/btc` followed by what you want:

| Command | What it does |
|---------|-------------|
| `/btc strategy.com` | Company detail card |
| `/btc Marathon` | Search by name |
| `/btc usa` | US companies leaderboard |
| `/btc top 10 mining` | Top 10 miners |
| `/btc whales` | Whale-tier companies |
| `/btc report etf` | ETF treasury analysis |
| `/btc countries` | Countries ranked by BTC |
| `/btc tiers` | Tier definitions |
| `/btc weather` | Global market weather (beta) |
| `/btc map` | ASCII world map |

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

### ASCII World Map

```
/btc map
```

```
 BITCOIN TREASURY WORLD MAP

          . _..::__:  ,-"-"._       |]       ,     _,.__
  _.___ _ _<_>`!(._`.`-.    /        _._     `_ ,_/  '  '-._.---.-..__
.{     " " `-==,',._\{  \  / {)     / _ ">_,-' `                 /-/_
 \_.:--.       `._ )`^-. "'      , [_/(                       __,/-'
'"'     \         "    _L       |-_,--'                )     /. (|
         |           ,'         _)_.\\._<> {}              _,' /  '
   [1]   `.         /          [_/_'` `"(     [2]       <'}  )
          \\    .-. )          /   `-'"..' `:._          _) [3]
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

 [1] United States      4,810,110 BTC
 [2] Malta                702,478 BTC
 [3] South Korea          239,042 BTC

 TOP 10 BY BTC HOLDINGS
 --------------------------------------------------------------------------
  #1   US  United States       4,810,110 BTC  (221 companies)
  #2   MT  Malta                 702,478 BTC  (  3 companies)
  #3   KR  South Korea          239,042 BTC  ( 10 companies)
  ...
 --------------------------------------------------------------------------
 View on site: https://bitcoincompanies.co/countries
```

Top 3 countries are placed on the map using pre-computed Mercator projection coordinates. Map from [Terminal-World-Map](https://github.com/HenrySeed/Terminal-World-Map).

## API

These skills use the public [Bitcoin Companies API](https://bitcoincompanies.co/api-docs):

- Base URL: `https://bitcoincompanies.co/api/v1`
- Auth: None required
- Rate limit: 60 requests/min per IP
- Format: JSON (Stripe-style responses)

## License

MIT
