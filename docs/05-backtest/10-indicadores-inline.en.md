# Inline Indicators in Backtest

> `indicadores_receitas` — declare indicators as Rhai recipes without pre-materializing.

Instead of creating a pipeline, materializing and passing the URI in `indicadores`, you can declare indicators inline directly in the backtest call:

## Syntax

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"sma_trend\"][0] > ind[\"sma_trend\"][1] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" },
      "sma_trend": { "receita": "sma(close, 50)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001
  }
}
```

The tool materializes each recipe over the series and injects as `ind["alias"]`.

## When to use inline vs pre-materialized

| Situation | Use |
|---|---|
| Quick backtest, few indicators | `indicadores_receitas` (inline) |
| Reuse indicators in multiple backtests | Pipeline → `indicadores` (URI) |
| Complex indicators (multi-step pipeline) | Pipeline → `indicadores` (URI) |

---

> Next: [The simulator](./11-simulador.en.md)
