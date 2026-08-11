# L2 Data Query

> `consultar_trades` and `consultar_book` — aggregated microstructure query.

These tools return **URI + meta + bounded statistics** — the model reads data point-by-point via resources, not via the tool.

## `consultar_trades`

```json
{
  "name": "consultar_trades",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/trades_1s", "agregacao": "1h" }
}
```

Returns: `{ uri, agregacao, row_count, first_ts, last_ts, buckets, summary }`

## `consultar_book`

```json
{
  "name": "consultar_book",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s", "agregacao": "5m" }
}
```

## Point-by-point reading

The model reads data via resource templates:

```
ct://series/binance/BTCUSDT/trades_1s/tail/100
ct://series/binance/BTCUSDT/book_1s/tail/100
```

---

> Next: [Microstructure indicators](./04-indicadores-micro.en.md)
