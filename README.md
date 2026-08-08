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

One real, unmodified UTC day — **2026-07-24** — of the BTC 5-minute series plus
the BTC settlement price feed:

| file | rows | what |
|---|---|---|
| `BTCUSD-prices-2026-07-24.csv.gz` | 78,515 | settlement price ticks |
| `BTC-5m-book-2026-07-24.jsonl.gz` | 157,588 | order-book snapshots |
| `BTC-5m-price_change-2026-07-24.jsonl.gz` | 238,058 | order-book deltas (best bid/ask) |
| `BTC-5m-last_trade_price-2026-07-24.jsonl.gz` | 605,976 | every trade |
| `BTC-5m-markets-2026-07-24.jsonl.gz` | 292 | markets + outcomes + strikes |

Field-level documentation: [`DATA_GUIDE.md`](DATA_GUIDE.md) (English) /
[`数据使用说明.md`](数据使用说明.md) (中文).

## Capture latency / 采集延迟

Timestamps are only worth what the capture path is worth, so here is ours,
measured on the sample day. `recv_ms` is our receive time; the reference is the
timestamp the upstream itself put on the message.

| stream | p50 | p95 |
|---|---|---|
| CLOB order book (`recv_ms − event_ts_ms`) | **9 ms** | 37 ms |
| Chainlink settlement feed (`recv_ms − server_ts_ms`) | **286 ms** | 433 ms |

Collection runs next to the venues' own infrastructure (`eu-west-1`). Moving
there cut settlement-feed latency from 383 ms to 286 ms at p50 — every tick in
this dataset carries all three timestamps, so you can verify the capture path
yourself rather than take our word for it.

采集点部署在 `eu-west-1`，紧邻场方基础设施；迁移后结算价流延迟 p50 从 383ms
降到 286ms。每条 tick 都带三个时间戳，延迟链路可自行核验。

## Verify it yourself / 自行验证

The point of a settlement feed is that you can re-derive the outcome from it.
Here is that check run against these exact files. (The sample day 2026-07-24
predates the 2026-08-07 TWAP switch, so the rule shown below is the one that
settled these markets; markets from 2026-08-07 onward are verified the same
way against the `twap` files instead. 样例日早于 TWAP 切换，下述验证使用当时
的即时价规则；08-07 起的市场以同样方法对 `twap` 文件验证。)

Settlement rule: **Up wins when the price at the close is greater than *or equal
to* `strike_value`** (the official market rules say "greater than or equal to",
so a tie settles Up). The feed is ~1Hz and whole-second aligned, so the report
carrying the closing second is normally present.

On 2026-07-24 the BTC 5-minute series had **288 settled binary markets**:

| | count | result |
|---|---|---|
| we hold the exact closing-second report | 257 | **257 of 257 reproduce the official outcome** |
| closing second missing, nearest tick 1–55s earlier | 13 | 11 reproduce; 2 differ |
| nearest tick older than 60s (close fell in a feed gap) | 1 | not independently reproducible |
| no `strike_value` (no tick at the opening second) | 17 | cannot be recomputed |

The two that differ are not data errors and we do not paper over them: in both,
the exact closing-second report is missing from the feed and the price was
sitting on the strike, so a neighbouring tick simply is not proof of which side
the close landed on. A settlement is only independently *provable* when we hold
the exact closing-second report — and for every one of the 257 markets where we
do, our recomputation matches Polymarket. The 4 remaining rows in the file are
markets straddling UTC midnight; they settle in the next day's file.

结算规则：**收盘价 ≥ `strike_value` 时 Up 赢**（官方规则原文 "greater than or
equal to"，持平判 Up）。2026-07-24 当天 BTC 5 分钟局共 288 个已结算二元市场：持有
**精确收盘秒**报价的 257 个，**257/257 全部与官方结果一致**；收盘秒缺失、用 1–55
秒前邻近 tick 推断的 13 个中 11 个一致、2 个不同；1 个收盘落在断档窗（最近 tick
超过 60 秒）无法独立复现；17 个起点秒无 tick 故无 strike。那 2 个不同的并非数据
错误：收盘秒报价缺失且价格正贴着 strike，邻近 tick 本就不足以证明收盘落在哪一侧
——我们不掩饰，也不用邻居去附会。只有持有精确收盘秒报价才谈得上独立**证明**，而
这 257 个全部对得上。文件里另外 4 行是跨 UTC 午夜的局，在次日文件里结算。

## Coverage on the sample day / 样例当天的覆盖情况

We publish gaps rather than hide them, because a dataset you cannot trust the
gaps of is a dataset you cannot backtest on.

On 2026-07-24 the settlement feed delivered a report for **78,515 of the day's
86,400 seconds (90.9%)**, in a tick pattern of ~1Hz. The missing time is **38
windows longer than 60 seconds, totalling about 73 minutes**, the longest being
387 seconds. We traced the largest window (11:00–11:07 UTC) directly: throughout
it the Binance mirror topic on the *same* WebSocket connection kept delivering
at its normal rate and our order-book feeds kept flowing, while the Chainlink
topic published nothing — an upstream publisher gap, not a capture failure.
Feed density is an upstream property and it drifts over time: 2026-06-08, our
first full day, ran at 97.9%.

The paid dataset ships a machine-readable **coverage report** for every day
listing tick-gap percentiles and every gap window, so you can exclude affected
periods from a backtest instead of discovering them the hard way.

2026-07-24 当天结算价流覆盖 86,400 秒中的 **78,515 秒（90.9%）**；缺失部分为 **38
段超过 60 秒的窗口、合计约 73 分钟**，最长 387 秒。我们直接追查了其中最长的一段
（UTC 11:00–11:07）：整段期间**同一条 WebSocket 连接**上的币安镜像 topic 保持正常
速率、我们的盘口流也持续在流，唯独 Chainlink topic 一条未发——属上游发布方断供，
而非采集失败。feed 密度属上游特性且会随时间漂移：首个完整日 2026-06-08 为 97.9%。
付费数据集每天附带机器可读的 **coverage 报告**（tick 间隔分位数 + 每个断档窗口），
可用于在回测中直接剔除受影响时段。

## Buy / 购买

Telegram: **@hankson_level** — delivery is an expiring private download link
(tar bundle with checksums and the data guide), scoped to exactly the assets,
data types and date range you purchase.
