# Backtest ↔ ML Bridge

> `avalar_backtest` — evaluate the model economically, not just statistically.

`montar_esteira_ml` accepts `avalar_backtest` in the config. This runs a `ct_backtest` over the materialized prediction:

```json
{
  "avalar_backtest": {
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }"
  }
}
```

The result includes `bt_*` metrics (bt_pnl, bt_sharpe, bt_drawdown, etc.) and `backtest_uri`.

> Default strategy: long if pred > 0, short if pred < 0.

---

> Next: [`uv` environment](./15-uv.en.md)
