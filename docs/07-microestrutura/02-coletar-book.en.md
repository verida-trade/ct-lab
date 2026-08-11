# Order Book Collection (1 second)

> **Premium.** `coletar_book` — reconstruct the order book locally and aggregate to 1 second.

## Call

```json
{
  "name": "coletar_book",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT" }
}
```

> Book has no `backfill_dias` — Spot has no bulk L2; Futures `bookDepth/` is aggregated ±%, doesn't reconstruct.

## `book_1s` schema (51 columns)

Same 40 columns as `trades_1s` (adapted) + 11 book-only extras.

### Mapping (40 symmetric columns)

| `trades_1s` | `book_1s` |
|---|---|
| `n_trades` | `n_updates` |
| `n_buys`/`n_sells` | `n_placements`/`n_cancellations` |
| `qty` | `qty_change_total` |
| `price_o/h/l/c` | `mid_o/h/l/c` |

### Book-only extras (11)

`bid_price`, `ask_price`, `bid_qty_top`, `ask_qty_top`, `spread_bps`, `bid_qty_delta`, `ask_qty_delta`, `depth_0_1pct_bid`, `depth_0_1pct_ask`, `depth_1pct_bid`, `depth_1pct_ask`

## Empty seconds (book)

State cols carry last value (don't become NaN — would mean "book disappeared"). `mid_o/h/l/c` = previous `mid_close`.

## URI

```
ct://series/binance/BTCUSDT/book_1s
```

---

> Next: [Data query](./03-consulta.en.md)
