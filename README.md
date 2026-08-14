# Polymarket Crypto Up/Down — Market Data

Tick-level market data for Polymarket crypto Up/Down markets, collected 24/7
directly from the source feeds. This repository hosts **free samples and
documentation**; the full dataset is sold by subscription or by range.

本仓库提供 Polymarket 加密 Up/Down 市场数据的**免费样例与文档**；完整数据集
按订阅或按区间出售。

> Also available: **[Predict.fun data](https://github.com/Ligengxin96/predict.fun-data-samples)**
> — the prediction-market venue behind the Binance Wallet front end, collected
> and sold separately (different venue, different settlement source).
>
> 另有 **[Predict.fun 数据](https://github.com/Ligengxin96/predict.fun-data-samples)**
> （币安钱包前端接入的预测市场），独立采集、独立出售。

## What the full dataset contains / 完整数据集内容

| dataset | description |
|---|---|
| `prices` | The instantaneous Chainlink Data Streams feed, tick-by-tick (~1Hz per symbol), full-precision values, three independent timestamps per tick. **Settled the markets through 2026-08-06**; still the underlying price line |
| `twap` | The **Chainlink TWAP streams that settle the markets since 2026-08-07** — 30s-lookback stream for 5-minute markets, 60s for 15-minute (~1Hz, full precision, same columns as `prices`) |
| `book` | Full-depth CLOB order-book snapshots, up to 1/sec per token |
| `price_change` | Order-book deltas with best bid/ask, sub-second |
| `last_trade_price` | **Every** trade print — never sampled or throttled |
| `markets` | Per-market metadata with **settlement outcome** (who won) and **strike** (the official priceToBeat) |

- Assets: BTC, ETH, SOL, DOGE, XRP, BNB, HYPE (ZEC since 2026-08) × intervals 5m / 15m
- History from 2026-06-06, growing daily; TWAP streams from 2026-08-08
  (first complete UTC day 2026-08-09)
- Every file ships with row counts + SHA-256; a daily **coverage report**
  discloses every gap honestly — nothing is hidden

### Settlement source change on 2026-08-07 / 结算源切换（2026-08-07）

From 2026-08-07 00:00 UTC, Polymarket settles its crypto Up/Down markets on
Chainlink **TWAP** (time-weighted average price) streams instead of the
instantaneous feed: the market resolves Up when the TWAP value at the close is
greater than or equal to the TWAP value at the open (30s lookback for 5-minute
markets, 60s for 15-minute; each market carries this in
`raw.cryptoMarketConfig`). **To recompute outcomes, use the `twap` files for
markets from 2026-08-07 onward and the `prices` files for earlier markets.**
The TWAP streams cannot be reconstructed exactly from the ~1Hz instantaneous
ticks — Chainlink computes them from its internal higher-frequency data — which
is why the dataset carries both lines. 2026-08-07 itself predates our TWAP
collection; for that single day the official outcome labels in the `markets`
files are the settlement authority.

2026-08-07 00:00 UTC 起，Polymarket 加密 Up/Down 市场改用 Chainlink **TWAP**
（时间加权均价）流结算：收盘 TWAP 值 ≥ 开盘 TWAP 值判 Up（5 分钟市场用 30 秒回看，
15 分钟用 60 秒；每个市场的 `raw.cryptoMarketConfig` 里带此标记）。**重算输赢：
2026-08-07 起的市场用 `twap` 文件，此前的市场用 `prices` 文件。** TWAP 流无法从
约 1Hz 的瞬时 tick 精确重建（Chainlink 用其内部更高频数据计算），因此数据集同时
提供两条线。2026-08-07 当天早于我们的 TWAP 采集起点，该天以 `markets` 文件中的
官方结算标签为准。

## Samples / 样例

**Easiest download: [Releases → samples-v1](https://github.com/Ligengxin96/polymarket-data-samples/releases/tag/samples-v1)**
(the same files also live in `samples/` for browsing).

One real, unmodified UTC day — **2026-08-12** — of the BTC 5-minute series plus
all three BTC settlement price lines, including the two TWAP streams that
settle the markets today:

| file | rows | what |
|---|---|---|
| `BTCUSD-twap30s-prices-2026-08-12.csv.gz` | 76,156 | **TWAP 30s stream — settles the 5-minute markets** |
| `BTCUSD-twap60s-prices-2026-08-12.csv.gz` | 76,162 | **TWAP 60s stream — settles the 15-minute markets** |
| `BTCUSD-prices-2026-08-12.csv.gz` | 76,131 | instantaneous Chainlink feed |
| `BTC-5m-book-2026-08-12.jsonl.gz` | 145,917 | order-book snapshots |
| `BTC-5m-price_change-2026-08-12.jsonl.gz` | 274,896 | order-book deltas (best bid/ask) |
| `BTC-5m-last_trade_price-2026-08-12.jsonl.gz` | 455,215 | every trade |
| `BTC-5m-markets-2026-08-12.jsonl.gz` | 292 | markets + outcomes + strikes |

Row counts are data rows (CSV headers excluded); every file carries its SHA-256
in [`samples/manifest.json`](samples/manifest.json).

Field-level documentation: [`DATA_GUIDE.md`](DATA_GUIDE.md) (English) /
[`数据使用说明.md`](数据使用说明.md) (中文).

## Capture latency / 采集延迟

Timestamps are only worth what the capture path is worth, so here is ours,
measured on the sample day. `recv_ms` is our receive time; the reference is the
timestamp the upstream itself put on the message.

| stream | p50 | p95 |
|---|---|---|
| CLOB order book (`recv_ms − event_ts_ms`) | **10 ms** | 19 ms |
| Chainlink TWAP 60s stream (`recv_ms − server_ts_ms`) | **223 ms** | 352 ms |
| Chainlink TWAP 30s stream (`recv_ms − server_ts_ms`) | **265 ms** | 402 ms |
| Chainlink instantaneous feed (`recv_ms − server_ts_ms`) | **281 ms** | 427 ms |

Collection runs next to the venues' own infrastructure (`eu-west-1`). Every
tick in this dataset carries all three timestamps, so you can verify the
capture path yourself rather than take our word for it.

采集点部署在 `eu-west-1`，紧邻场方基础设施。每条 tick 都带三个时间戳，延迟链路
可自行核验。

## Verify it yourself / 自行验证

The point of a settlement feed is that you can re-derive the outcome from it.
Here is that check, run against these exact files — nothing but this repository
is needed to reproduce it.

Every market in this sample carries `raw.cryptoMarketConfig.twapEnabled = true`
with a 30-second lookback, so the governing rule is the current one: **Up wins
when the TWAP value at the close is greater than or equal to the TWAP value at
the open.** Both values are read from
`BTCUSD-twap30s-prices-2026-08-12.csv.gz` at the exact boundary seconds, using
the full-precision integer column (`full_accuracy_value`) — no floating point
anywhere in the comparison.

On 2026-08-12 the BTC 5-minute series had **288 markets settling inside the
day** (the file's other 4 rows close after midnight and settle in the next
day's file):

| | count | result |
|---|---|---|
| we hold the exact TWAP reports at both boundary seconds | 222 | **222 of 222 reproduce the official outcome** |
| a boundary second is not present in this file | 66 | not independently provable — reported as undetermined |

We call a settlement reproducible only when we hold the exact boundary-second
reports on **both** ends. A neighbouring tick is not proof of where the
boundary actually landed, so those markets are reported as undetermined rather
than counted as agreement — and of the 222 where we do hold both ends, every
single one matches Polymarket. (One of the 66 is simply a market that opened in
the previous day's file.)

本样例中每个市场的 `raw.cryptoMarketConfig.twapEnabled` 均为 `true`、回看 30 秒，
因此适用当前规则：**收盘时刻 TWAP 值 ≥ 开盘时刻 TWAP 值则 Up 赢**。两端取值均来自
`BTCUSD-twap30s-prices-2026-08-12.csv.gz` 中精确边界秒的全精度整数列
（`full_accuracy_value`），比较过程全程不使用浮点。

2026-08-12 当天 BTC 5 分钟局共 **288 个在本日内结算的市场**（文件中另外 4 行收盘在
午夜之后，于次日文件结算）：两端都持有精确边界秒 TWAP 报价的 **222 个，222/222
全部与官方结果一致**；另有 66 个因某一端边界秒不在本文件内，按不可独立证明处理、
不作判定（其中 1 个只是开盘落在前一日文件里）。

只有两端都持有精确边界秒报价，我们才称其可复现——邻近 tick 并不能证明边界究竟落在
哪一侧，因此这类市场记为未判定而非算作一致。而这 222 个全部对得上。

## Coverage reporting / 覆盖情况报告

Feed density is a property of the upstream publisher, not something a collector
can invent — so every day of the paid dataset ships a machine-readable
**coverage report** alongside the data: per-symbol row counts, tick-gap
percentiles (p50/p95/max) and the day's largest gap windows. You can size and
locate the affected periods up front instead of discovering them mid-backtest,
and you never have to take a completeness claim on trust.

付费数据集每天随数据附带机器可读的 **coverage 报告**：每个币种的行数、tick 间隔
分位数（p50/p95/max）与当日最大的若干个断档窗口。feed 密度属上游发布方特性，报告
让你在回测前就能定位并评估受影响时段，而不必对完整性声明照单全收。

## Buy / 购买

Telegram: **@hankson_level** — delivery is an expiring private download link
(tar bundle with checksums and the data guide), scoped to exactly the assets,
data types and date range you purchase.
