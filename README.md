<p align="center"><a href="https://apify.com/DevilScrapes/binance-market-data-scraper"><img src="assets/banner.svg" alt="Binance Market Data Scraper — Ticker &amp; Klines — Python example" width="100%"></a></p>

# Binance Market Data Scraper in Python: spot tickers and OHLCV klines into pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/binance-market-data-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/binance-market-data-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/binance-market-data-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Here you get a tested, deliberately small Python wrapper around the **[Binance Market Data Scraper](https://apify.com/DevilScrapes/binance-market-data-scraper)** on Apify. Name the spot pairs you care about, say `BTCUSDT` and `SOLUSDT`, pick a candle size, and the script hands back two kinds of rows: one rolling 24-hour summary per pair and a sequence of open-high-low-close-volume candles. The requests to Binance happen on Apify's side, so your laptop needs neither a Binance account nor an exchange key, and a cron job on a small box can collect the same data every night. Everything lands in `results.json` and `results.csv`, which a notebook turns into a DataFrame in one line.

> **Actor:** [Binance Market Data Scraper — Ticker & Klines](https://apify.com/DevilScrapes/binance-market-data-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [binance-market-data-scraper-javascript](https://github.com/devilscrapes/binance-market-data-scraper-javascript)

## 🔥 What you get

- **Two shapes, one dataset**: `rowType` says whether a row is a `ticker` snapshot covering the trailing 24 hours or a single `kline` candle, so one filter splits a run into a snapshot table and a time series.
- **Up to 50 pairs per run**: the 24h stats for your whole list come from one batched request, and candles are fetched per symbol, five symbols at a time.
- **Volume ranking when you skip the list**: leave `symbols` empty with ticker-only output and the run returns the busiest pairs by 24h quote volume, capped by `maxSymbols` (1 to 100).
- **Eight candle sizes**: `1m` through `1w`, with up to 1,000 candles per symbol and optional `startTime` and `endTime` bounds in epoch milliseconds.
- **Numbers arrive as numbers**: Binance's quoted decimal strings are converted to floats and its millisecond timestamps become ISO-8601 UTC strings, so nothing needs casting before arithmetic.
- **Careful with bans**: transient errors are retried with backoff and `Retry-After` is respected, but a `418` ban or `451` location refusal ends the run at once instead of retrying into a longer block.
- **Routing that clears the location check**: by default requests leave through a residential proxy exit outside the US, because Binance's main host refuses US traffic; your own proxy group or country overrides that.
- **One dead pair stays one dead pair**: a symbol whose candle request fails is logged and skipped, and the run only fails when every requested symbol failed.

## 🎯 Use cases

### A clean candle file for backtests

Request `1h` or `1d` klines for the pairs in your research universe, filter on `rowType == "kline"`, and pivot `close` by `openTime`. You get a wide price matrix with UTC timestamps that a backtesting notebook can read directly, without first decoding Binance's positional arrays.

### Volatility and range studies

Compute log returns per symbol with a `groupby`, then compare their standard deviation across `15m`, `4h` and `1d` runs. The per-candle `high` and `low` also give you true ranges, so a study of how noisy each pair is takes one cell of pandas.

### Order-flow ratios from taker volume

Each candle carries `takerBuyBaseVolume` next to `volume`, so the share of volume where the buyer was the aggressor is a single division. Plot it against price moves to see whether it tells you anything for your pairs; the dataset makes the question cheap to ask.

### A nightly snapshot of the busiest pairs

Run ticker-only with an empty `symbols` list from cron, and append each night's CSV to a dated folder. After a month you have a history of which pairs led on 24h quote volume and how their trade counts changed, which Binance itself does not keep for you.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/binance-market-data-scraper-python.git
cd binance-market-data-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Binance Market Data Scraper — Ticker & Klines](https://apify.com/DevilScrapes/binance-market-data-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Stripped to the essentials, the integration fits in a few lines. The `apify-client` package starts the Actor, blocks until the run ends and walks the dataset page by page.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/binance-market-data-scraper").call(run_input={'symbols': ['BTCUSDT'], 'dataTypes': ['ticker', 'klines'], 'interval': '1d', 'klinesLimit': 9, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` forwards `input.json` to the Actor unchanged. The bundled example asks for one pair, one 24h ticker row and nine daily candles, ten rows in total, so a first test costs a few cents. Add pairs to `symbols`, switch `interval` and raise `klinesLimit` once the output looks right; set `startTime` and `endTime` when you need a specific historical window.

```json
{
  "symbols": [
    "BTCUSDT"
  ],
  "dataTypes": [
    "ticker",
    "klines"
  ],
  "interval": "1d",
  "klinesLimit": 9,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `symbols` | array | no | `["BTCUSDT", "ETHUSDT", "SOLUSDT"]` | — |
| `maxSymbols` | integer | no | `20` | — |
| `dataTypes` | array | no | `["ticker", "klines"]` | — |
| `interval` | string | no | `"1h"` | `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, `1w` |
| `klinesLimit` | integer | no | `100` | — |
| `startTime` | integer | no | — | — |
| `endTime` | integer | no | — | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Ticker rows fill the 24h fields (`lastPrice`, `priceChangePercent`, `weightedAvgPrice`, `bidPrice`, `askPrice`, `quoteVolume`, `tradeCount`) and leave the candle fields as `None`; kline rows do the reverse with `open`, `high`, `low`, `close`, `quoteAssetVolume` and the taker volumes. Both share `symbol`, `volume`, `openTime` and `closeTime`. Filter on `rowType` before you aggregate, or a mean over `volume` will mix a day's total with single candles.

| Field | Type | Always present |
|---|---|---|
| `rowType` | string | yes |
| `symbol` | string or null | no |
| `priceChange` | number or null | no |
| `priceChangePercent` | number or null | no |
| `weightedAvgPrice` | number or null | no |
| `lastPrice` | number or null | no |
| `bidPrice` | number or null | no |
| `askPrice` | number or null | no |
| `openPrice` | number or null | no |
| `highPrice` | number or null | no |
| `lowPrice` | number or null | no |
| `volume` | number or null | no |
| `quoteVolume` | number or null | no |
| `tradeCount` | integer or null | no |
| `interval` | string or null | no |
| `open` | number or null | no |
| `high` | number or null | no |
| `low` | number or null | no |
| `close` | number or null | no |
| `quoteAssetVolume` | number or null | no |
| `numberOfTrades` | integer or null | no |
| `takerBuyBaseVolume` | number or null | no |
| `takerBuyQuoteVolume` | number or null | no |
| `openTime` | string or null | no |
| `closeTime` | string or null | no |
| `scrapedAt` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Ticker row emitted | $0.004 |
| Kline row emitted | $0.0015 |

Prices as configured on 2026-09-10. The [Actor page](https://apify.com/DevilScrapes/binance-market-data-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

The pandas script below (`pip install pandas`) separates the two row types, adds a volume-weighted price and a taker buy share per candle, summarises each symbol, and writes a wide matrix of closing prices you can hand to a backtest.

```python
import json

import numpy as np
import pandas as pd

rows = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))

candles = rows[rows["rowType"] == "kline"].copy()
candles["openTime"] = pd.to_datetime(candles["openTime"])
candles = candles.sort_values(["symbol", "openTime"])
candles["vwap"] = candles["quoteAssetVolume"] / candles["volume"]
candles["taker_buy_share"] = candles["takerBuyBaseVolume"] / candles["volume"]
candles["log_return"] = candles.groupby("symbol")["close"].transform(lambda s: np.log(s).diff())

per_symbol = candles.groupby("symbol").agg(
    candles=("openTime", "count"),
    first_open=("openTime", "min"),
    last_close=("close", "last"),
    window_high=("high", "max"),
    window_low=("low", "min"),
    mean_taker_buy_share=("taker_buy_share", "mean"),
    return_stdev=("log_return", "std"),
)
per_symbol.to_csv("kline_summary.csv")

closes = candles.pivot(index="openTime", columns="symbol", values="close")
closes.to_csv("closes_wide.csv")

snapshot = rows[rows["rowType"] == "ticker"][["symbol", "lastPrice", "priceChangePercent", "quoteVolume", "tradeCount"]]
print(snapshot.sort_values("quoteVolume", ascending=False).to_string(index=False))
print(per_symbol.to_string(float_format=lambda v: f"{v:.4f}"))
```

## ❓ FAQ

### Do I need a Binance account or API key?

No. The Actor only calls Binance's public spot market data endpoints, which take no signature. Your Apify token is the single secret in this setup.

### How do I get more than 1,000 candles for one pair?

One run makes a single candle request per symbol, so `klinesLimit` tops out at 1,000. For a longer history, loop over consecutive `startTime` and `endTime` windows in Python, start one run per window and concatenate the DataFrames, dropping duplicate `openTime` values at the joins.

### Are prices exact enough for accounting?

They are double-precision floats, which is fine for charts, statistics and research. If you reconcile balances to the last satoshi, convert the values you need with `decimal.Decimal(str(x))` and treat Binance's own records as the reference.

### Does it cover futures, margin or my account?

No. It reads spot market data only. Futures and options markets, order books, private balances and order history are out of scope.

### Is this a trading signal or investment advice?

Neither. It delivers raw market data for research, reporting and engineering work. What you conclude from it, and any trading decision, is your own responsibility.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/binance-market-data-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Binance Market Data Scraper — Ticker & Klines on Apify Store](https://apify.com/DevilScrapes/binance-market-data-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/binance-market-data-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [CoinGecko API Alternative — Crypto Scraper](https://apify.com/DevilScrapes/coingecko-crypto-scraper) — market cap, supply and all-time highs across coins rather than per exchange pair.
- [CoinMarketCap Crypto Scraper — Ranked Listing](https://apify.com/DevilScrapes/coinmarketcap-crypto-scraper) — the ranked coin listing with tags and supply figures.
- [Dexscreener Token & Pair Data Scraper](https://apify.com/DevilScrapes/dexscreener-scraper) — on-chain DEX pairs with liquidity and buy/sell counts.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/binance-market-data-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Binance.
