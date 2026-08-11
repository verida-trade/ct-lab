# Trades Collection (1 second)

> **Premium.** `coletar_trades` — collect trade data in real-time with 1-second granularity.

## Call

```json
{
  "name": "coletar_trades",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "backfill_dias": 7 }
}
```

| Parameter | Description |
|---|---|
| `provider` | `"binance"` (Spot) or `"binance_um"` (Futures USDⓈ-M) |
| `symbol` | Symbol (e.g., `"BTCUSDT"`) |
| `backfill_dias` | N days of bulk backfill (optional; downloads daily dumps from Binance) |

## Return

```json
{ "collector_id": "binance:BTCUSDT:trades_1s", "status": "started", "started_at": 1784951100 }
```

## `trades_1s` schema (41 columns)

| Group | Columns |
|---|---|
| Count (4) | `n_trades`, `n_buys`, `n_sells`, `n_delta` |
| Base volume (4) | `qty`, `buy_qty`, `sell_qty`, `qty_delta` |
| Quote volume (4) | `quote_qty`, `buy_quote_qty`, `sell_quote_qty`, `quote_delta` |
| Price (5) | `vwap`, `price_open`, `price_high`, `price_low`, `price_close` |
| Cursor (2) | `first_agg_id`, `last_agg_id` |
| Qty distribution (11) | min, max, mean, std, p01, p10, p25, p50, p75, p90, p99 |
| Quote distribution (11) | same |

## Empty seconds

Seconds with no trades: `n_*=0`, volumes=0, prices=NaN, cursors=NaN. **Line emitted normally** — explicit gaps.

## URI

```
ct://series/binance/BTCUSDT/trades_1s
```

---

> Next: [Book collection](./02-coletar-book.en.md)
