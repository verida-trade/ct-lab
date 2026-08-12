# Recipe 03 — RSI with ADX Filter

> **Level:** Intermediate · **Prerequisites:** [Recipe 01](./01-cruzamento-sma.en.md)

Buy when RSI < 30 **AND** ADX > 25 (strong trend). Sell when RSI > 70. The ADX filter eliminates reversal signals in choppy markets.

> ⚠️ **ADX returns a map** (`adx`, `plus_di`, `minus_di`), not a single series. Therefore it **does NOT work** as `indicadores_receitas` — you must materialize it via a pipeline with `compose`.

---

## Step 1 — Fetch series

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

## Step 2 — Materialize ADX + RSI via pipeline

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_adx_rsi",
    "output": "$concat",
    "steps": [
      { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
      { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
      {
        "id": "concat",
        "op": "compose",
        "columns": [
          { "source": "$adx", "source_column": "adx", "as_column": "adx" },
          { "source": "$rsi", "source_column": "rsi", "as_column": "rsi" }
        ]
      }
    ]
  }
}
```

The series `ct://derived/btc_adx_rsi` contains `adx` and `rsi` aligned bar by bar.

## Step 3 — Backtest with filter

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_adx_rsi",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_adx_filter"
  }
}
```

## Real results

| Variant | Trades | PnL Total | PnL Gross | Fees | Win% | PF |
|---|---|---|---|---|---|---|
| Pure RSI (no fee) | 135 | +$1,496 | +$1,496 | $0 | 73.3% | 1.228 |
| Pure RSI (0.1% fee) | 135 | -$15,834 | +$1,496 | $17,330 | 9.6% | 0.099 |
| RSI+ADX>25 (no fee) | 72 | -$147 | -$147 | $0 | 63.9% | 0.963 |
| RSI+ADX>25 (0.1% fee) | 72 | -$9,381 | -$147 | $9,234 | 9.7% | 0.066 |
| RSI+ADX>30 (0.1% fee) | 53 | -$5,902 | +$880 | $6,783 | 17.0% | 0.120 |

### Interpretation

- **The filter reduces trades** (135→72) — good for fees, which drop from $17,330 to $9,234.
- **But it also reduces gross edge** (from +$1,496 to –$147) — the filter eliminated more good signals than whipsaws.
- **ADX>30 is the best variant** with fees: fewer trades (53), positive gross edge (+$880), but still insufficient (edge/fee ratio = 0.13 ≪ 1.0).
- **A filter doesn't create edge** — it only removes signals. If some were good, you lost edge along with the noise.

## Variations

- **ADX as directional gate:** only buy if DI+ > DI− AND ADX > 25
- **Tighter RSI:** oversold 25, overbought 75
- **Threshold 30:** `ind["adx"][0] > 30.0` (more restrictive, fewer trades)
- **Add ATR for sizing:** `atr(high, low, close, 14)` and adjust lot by volatility
