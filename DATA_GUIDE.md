# Data Guide

(中文版见 数据使用说明.md)

## Layout

The sample bundle unpacks to a directory laid out exactly like the paid
archive, so anything written against a sample keeps working against a delivered
dataset unchanged:

```
polymarket-data-samples/
  data/polymarket/daily/markets/BTC-5m/BTC-5m-markets-<date>.jsonl.gz
  data/polymarket/daily/book/BTC-5m/BTC-5m-book-<date>.jsonl.gz
  data/polymarket/daily/best_bid_ask/BTC-5m/BTC-5m-best_bid_ask-<date>.jsonl.gz
  data/polymarket/daily/price_change/BTC-5m/BTC-5m-price_change-<date>.jsonl.gz
  data/polymarket/daily/last_trade_price/BTC-5m/BTC-5m-last_trade_price-<date>.jsonl.gz
  data/chainlink/daily/prices/BTCUSD/BTCUSD-prices-<date>.csv.gz
  data/chainlink-twap-30s/daily/prices/BTCUSD/BTCUSD-twap30s-prices-<date>.csv.gz
  data/chainlink-twap-60s/daily/prices/BTCUSD/BTCUSD-twap60s-prices-<date>.csv.gz
```

The directory segments carry meaning, and tooling reads them: `data/<venue>/daily/<dataset>/<ASSET>-<interval>/`
for market data, `data/<settlement stream>/daily/prices/<SYMBOL>/` for the
price lines. A full purchased dataset has the same shape with more assets,
more intervals and more days under it.

## <SYMBOL>-prices-<date>.csv.gz — Chainlink settlement price

| column | meaning |
|---|---|
| feed_ts_ms | price event time (ms, second-aligned) |
| value | price as float (convenience) |
| full_accuracy_value | exact price: integer string scaled by 1e18 — divide by 1e18 |
| server_ts_ms | relay server send time |
| recv_ms | collector receive time |

Note: two rows in the same second with different values = a same-second feed correction; the later recv_ms wins.

Excel users: full_accuracy_value exceeds Excel's 15-digit number limit and will display as scientific notation if you double-click the file. Either read the value column instead, or import via Data -> From Text/CSV and set the full_accuracy_value column type to Text.

## <SERIES>-markets-<date>.jsonl.gz — per-market metadata and settlement outcome

| field | meaning |
|---|---|
| slug | market id; suffix = slot start (unix sec) |
| start_sec / end_sec | slot boundaries (unix sec) |
| interval_sec | 300 = 5-minute market, 900 = 15-minute |
| token_ids | CLOB token ids, [Up, Down] order |
| resolved | settlement label present |
| outcome_prices | ["1","0"] Up won, ["0","1"] Down won, ["0.5","0.5"] split |
| strike_value | priceToBeat: integer string scaled by 1e18; null when no tick existed at the start second |
| raw | full Gamma API market object |

Note: settlement rule **for markets through 2026-08-06** (before the TWAP switch below) = Up wins iff the latest feed tick at or before end_sec (the value in effect at the close; the feed runs ~1Hz, so it is not always exactly on end_sec) is greater than **or equal to** strike_value — the official market rules read "greater than or equal to", so a tie settles Up. A few markets fall in feed gaps (no tick near end_sec, visible from the ticks' own timestamps) or have a null strike_value; those cannot be recomputed from the feed alone. Markets crossing UTC midnight appear in both days' files — dedupe by slug.

Settlement source change: markets from **2026-08-07 00:00 UTC** onward (those with `raw.cryptoMarketConfig.twapEnabled = true`) settle on the Chainlink **TWAP streams** instead — Up wins iff the TWAP stream's value at the close ≥ its value at the open.

**Which stream settles a market is written on the market itself**, in `raw.cryptoMarketConfig.twapLookbackSeconds` — read it per market rather than inferring it from the date, because upstream has moved it once already:

- from 2026-08-07: 5-minute markets settled on the 30s-lookback stream, 15-minute markets on the 60s stream
- from 2026-08-13 upstream began migrating 5-minute markets to the 60s stream, completing on 2026-08-14; **since 2026-08-15 every market, 5-minute and 15-minute alike, settles on the 60s stream**

The `twap30s` stream is still collected and still ships with every day of the dataset, but it no longer decides any market's outcome. The full dataset ships both as `twap30s`/`twap60s` price files (same columns as `prices`, coverage from 2026-08-08) — recompute post-switch markets from the stream the market's own config names, not from the instantaneous `prices` files.

## <SERIES>-book-<date>.jsonl.gz — full-depth order book snapshots

| field | meaning |
|---|---|
| slug | market |
| asset_id | token the snapshot belongs to (Up or Down) |
| event_ts_ms | CLOB frame time |
| recv_ms | collector receive time |
| payload.bids[] / payload.asks[] | all price levels, {price, size} strings |

Note: book state at time t = the token's latest snapshot with recv_ms <= t.

## <SERIES>-best_bid_ask-<date>.jsonl.gz — top of book, unthrottled

| field | meaning |
|---|---|
| slug | market |
| asset_id | the token this quote belongs to (each market has two) |
| event_type | always `best_bid_ask` in this file |
| event_ts_ms | **venue event time, epoch ms — use this one** |
| recv_ms | collector receive time |
| payload.best_bid / best_ask | best buy / sell price |
| payload.spread | best_ask - best_bid, as sent upstream |
| payload.timestamp | the venue's own event time, **as a string** |
| payload.market / asset_id / event_type | as sent upstream |

**Every value inside `payload` is a string**, including the timestamp. We
normalise the venue's event time into the top-level numeric `event_ts_ms`, so
read that instead of parsing `payload.timestamp` — measured over 541,310 frames
the two are identical on every single row.

The venue emits one of these whenever a token's top of book moves. Prices only —
there are **no sizes** here; for depth use `book` snapshots and `price_change`
deltas. The two legs of a market are normally pushed together (measured on
2026-09-08: 644,259 pairs against 2,579 single-leg pushes).

**An empty side is encoded as a price, not as null or a missing field**, and
because the two outcomes of a market always sum to 1, it shows up differently on
each leg: `best_bid = "0"` means *nobody is bidding*, and `best_ask = "1"` means
*nobody is offering below the cap*. They are the same event seen from the two
complementary tokens, and the counts prove it — on the sample day 11,043 frames
carry `best_bid = "0"` and exactly 11,043 carry `best_ask = "1"` (22,086 rows,
1.68%, no row has both).

So the domain to test against is the **closed interval [0, 1]**, not the
[0.001, 0.999] of a live quote. `"1"` is not a corrupt value here; treating it
as one discards the rows where the book state is least ambiguous.

**Why this file exists.** `price_change` already carries best_bid/best_ask on
every entry, but it is throttled, so a top of book rebuilt from deltas alone
skips moves. Counting a "move" as a change in the (best bid, best ask) pair for
one token, the same way on both files, the sample day holds **454,790** moves in
this file and only **288,298 — 63.4%** are recoverable from the deltas. BTC is
the best case (deltas kept at 1/20ms for BTC, 1/100ms for ETH, 1/500ms for
everything else); on other assets far less survives.

Availability: from 2026-09-02 (collection started 00:42:18 UTC that day, so it
is missing its first 42 minutes); the first complete UTC day is 2026-09-03.
Earlier days have no equivalent — `price_change` is the only top-of-book record
there, and it is throttled.

## <SERIES>-price_change-<date>.jsonl.gz — order-book deltas (best bid/ask)

| field | meaning |
|---|---|
| slug | market |
| event_ts_ms | CLOB frame time |
| recv_ms | collector receive time |
| payload.price_changes[] | one entry per changed level: {side, price, size, best_bid, best_ask, asset_id} |

Note: sub-second order-book deltas. Apply them in event_ts_ms order on top of the latest book snapshot to track best bid/ask between snapshots. Trades are never throttled.

Throttling (disclosed): frames are kept at most 1 per market per N ms (keep-first within each window), and N varies **by date and by asset**:

- from 2026-06-06: all assets 500ms
- from 2026-08-25: BTC 20ms, ETH 100ms; all others 500ms

So a BTC day from 2026-08-25 onward carries roughly 25x the deltas of an earlier one. If you reconstruct books across a date range that spans the change, expect the resolution to change with it.

## <SERIES>-last_trade_price-<date>.jsonl.gz — trade prints

Trades exactly as the venue's WebSocket broadcast them: stored unthrottled and never sampled, but not reconciled against on-chain fills.

| field | meaning |
|---|---|
| payload.price / size | trade price / size |
| payload.side | BUY = taker bought |
| payload.asset_id | traded token |
| payload.fee_rate_bps | fee rate (basis points) |
| payload.timestamp | trade time (ms) |
| recv_ms | collector receive time |

## samples/manifest.json — per-file row counts and sha256 checksums for this sample
