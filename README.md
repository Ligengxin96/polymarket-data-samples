# Polymarket Crypto Up/Down — Market Data

Tick-level market data for Polymarket crypto Up/Down markets, collected 24/7
directly from the source feeds. This repository hosts **free samples and
documentation**; the full dataset is sold by subscription or by range.

本仓库提供 Polymarket 加密 Up/Down 市场数据的**免费样例与文档**；完整数据集
按订阅或按区间出售。

## What the full dataset contains / 完整数据集内容

| dataset | description |
|---|---|
| `prices` | The Chainlink Data Streams feed that **settles** the markets, tick-by-tick (~1Hz per symbol), full-precision values, three independent timestamps per tick |
| `book` | Full-depth CLOB order-book snapshots, up to 1/sec per token |
| `price_change` | Order-book deltas with best bid/ask, sub-second |
| `last_trade_price` | **Every** trade print — never sampled or throttled |
| `markets` | Per-market metadata with **settlement outcome** (who won) and **strike** (the official priceToBeat) |

- Assets: BTC, ETH, SOL, DOGE, XRP, BNB, HYPE × intervals 5m / 15m
- History from 2026-06-06, growing daily
- Every file ships with row counts + SHA-256; a daily **coverage report**
  discloses every gap honestly — nothing is hidden

## Samples / 样例

**Easiest download: [Releases → samples-v1](https://github.com/Ligengxin96/polymarket-data-samples/releases/tag/samples-v1)**
(the same files also live in `samples/` for browsing).

One real, unmodified day (2026-06-08 UTC — our first complete 24-hour day) of
the BTC 5-minute series + the BTC settlement price feed:

| file | rows | what |
|---|---|---|
| `BTCUSD-prices-2026-06-08.csv.gz` | 84,558 | settlement price ticks |
| `BTC-5m-book-2026-06-08.jsonl.gz` | 171,055 | order-book snapshots |
| `BTC-5m-price_change-2026-06-08.jsonl.gz` | 249,496 | order-book deltas (best bid/ask) |
| `BTC-5m-last_trade_price-2026-06-08.jsonl.gz` | 820,730 | every trade |
| `BTC-5m-markets-2026-06-08.jsonl.gz` | 292 | markets + outcomes + strikes |

Field-level documentation: [`DATA_GUIDE.md`](DATA_GUIDE.md) (English) /
[`数据使用说明.md`](数据使用说明.md) (中文).

Verify the samples reproduce Polymarket's official settlement: for each
resolved market that carries a `strike_value`, take the latest `prices` tick
at or before `end_sec` (the feed runs ~1Hz; use the value in effect at the
close) and compare it against `strike_value` — Up wins iff it is higher,
matching `outcome_prices`. On 2026-06-08 this rule reproduces **285 of the
288** settled BTC 5-minute markets that carry a strike. The 3 it does not are
three consecutive post-midnight markets (ending 00:05 / 00:10 / 00:15 UTC)
that sit in a feed-coverage gap, where the last tick is 5–15 min before
`end_sec`; 4 further markets have a null `strike_value` (no tick at the start
second). These gaps are real and disclosed — nothing is smoothed over. The
full per-file coverage report (tick-gap percentiles and every gap window)
ships with the dataset.

## Buy / 购买

Telegram: **@hankson_level** — delivery is an expiring private download link
(tar bundle with checksums and the data guide), scoped to exactly the assets,
data types and date range you purchase.
