# Microstructure Indicators

> **Premium.** Indicators that operate on `trades_1s` or `book_1s`.

| Tool | Input | Window | What it measures |
|---|---|---|---|
| `tfi` | trades_1s | — | Trade Flow Imbalance (`qty_delta / qty`) |
| `ct_tfi` | trades_1s | `period` | VWMA of TFI weighted by qty |
| `bfi` | book_1s | — | Book Flow Imbalance |
| `ct_bfi` | book_1s | `period` | VWMA of BFI |
| `obi` | book_1s | — | Order Book Imbalance (top-of-book) |
| `ct_obi` | book_1s | `period` | VWMA of OBI |
| `dbi_01` | book_1s | — | Depth Book Imbalance bin ±0.1% |
| `dbi_1` | book_1s | — | Depth Book Imbalance bin ±1% |
| `mpo` | book_1s | — | Microprice Offset normalized by half-spread |

All return scale `[-1, +1]`. Empty window / no weight → `0`.

## Example

```json
{
  "name": "ct_tfi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/trades_1s", "period": 60 }
}
```

## Use in pipeline

Microstructure indicators can be used in the pipeline (`montar_pipeline_indicadores`) and in backtests the same way as price-based indicators.

---

> Next: [Collector status](./05-status.en.md)
