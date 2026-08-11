# Comparing Backtests

> Run multiple variations side by side and compare metrics.

## `ct_comparar`

Compares a strategy with different parameters in a single call:

```json
{
  "name": "ct_comparar",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < par[\"oversold\"] { comprado(1.0) } else if ind[\"rsi\"][0] > par[\"overbought\"] { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": { "rsi": { "receita": "rsi(close, 14)" } },
    "grid_parametros": {
      "oversold": [20.0, 25.0, 30.0],
      "overbought": [70.0, 75.0, 80.0]
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001
  }
}
```

This runs 9 backtests (3×3 grid) and returns a comparative table.

## `ct_buscar_backtests`

Lists previous backtests:

```json
{ "name": "ct_buscar_backtests", "arguments": { "limit": 10 } }
```

Returns the last 10 backtests with URI, name, summary metrics.

---

> Next: [Inline indicators](./10-indicadores-inline.en.md)
