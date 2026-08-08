# Data Guide

(中文版见 数据使用说明.md)

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

Note: settlement rule = Up wins iff the latest feed tick at or before end_sec (the value in effect at the close; the feed runs ~1Hz, so it is not always exactly on end_sec) is greater than **or equal to** strike_value — the official market rules read "greater than or equal to", so a tie settles Up. A few markets fall in disclosed feed-coverage gaps (no tick near end_sec) or have a null strike_value — see the coverage report; those cannot be recomputed from the feed alone. Markets crossing UTC midnight appear in both days' files — dedupe by slug.

Settlement source change: markets from **2026-08-07 00:00 UTC** onward (those with `raw.cryptoMarketConfig.twapEnabled = true`) settle on the Chainlink **TWAP streams** instead — Up wins iff the TWAP stream's value at the close ≥ its value at the open (30s-lookback stream for 5-minute markets, 60s for 15-minute). The full dataset ships those streams as `twap30s`/`twap60s` price files (same columns as `prices`, coverage from 2026-08-08) — recompute post-switch markets from them, not from the instantaneous `prices` files. The sample day 2026-07-24 predates the switch.

## <SERIES>-book-<date>.jsonl.gz — full-depth order book snapshots

| field | meaning |
|---|---|
| slug | market |
| asset_id | token the snapshot belongs to (Up or Down) |
| event_ts_ms | CLOB frame time |
| recv_ms | collector receive time |
| payload.bids[] / payload.asks[] | all price levels, {price, size} strings |

Note: book state at time t = the token's latest snapshot with recv_ms <= t.

## <SERIES>-price_change-<date>.jsonl.gz — order-book deltas (best bid/ask)

| field | meaning |
|---|---|
| slug | market |
| event_ts_ms | CLOB frame time |
| recv_ms | collector receive time |
| payload.price_changes[] | one entry per changed level: {side, price, size, best_bid, best_ask, asset_id} |

Note: sub-second order-book deltas. Apply them in event_ts_ms order on top of the latest book snapshot to track best bid/ask between snapshots. Throttled keep-first to 500ms per market (disclosed); trades are never throttled.

## <SERIES>-last_trade_price-<date>.jsonl.gz — every trade

| field | meaning |
|---|---|
| payload.price / size | trade price / size |
| payload.side | BUY = taker bought |
| payload.asset_id | traded token |
| payload.fee_rate_bps | fee rate (basis points) |
| payload.timestamp | trade time (ms) |
| recv_ms | collector receive time |

## manifest.json — file inventory with per-file row counts and sha256 checksums
