# Ponte Backtest ↔ ML

> `avalar_backtest` — avalie o modelo economicamente, não só estatisticamente.

A `montar_esteira_ml` aceita `avalar_backtest` na config. Isto roda um `ct_backtest` sobre a predição materializada:

```json
{
  "avalar_backtest": {
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }"
  }
}
```

O resultado inclui métricas `bt_*` (bt_pnl, bt_sharpe, bt_drawdown, etc.) e `backtest_uri`.

> A estratégia default é long se pred > 0, short se pred < 0.

---

> Próximo: [Ambiente `uv`](./15-uv.pt.md)
