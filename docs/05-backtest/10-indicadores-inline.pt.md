# Indicadores Inline no Backtest

> `indicadores_receitas` — declare indicadores como receitas Rhai sem pré-materializar.

Em vez de criar uma pipeline, materializar e passar a URI em `indicadores`, você pode declarar indicadores inline direto na chamada do backtest:

## Sintaxe

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"sma_trend\"][0] > ind[\"sma_trend\"][1] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" },
      "sma_trend": { "receita": "sma(close, 50)", "parametros": {} }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001
  }
}
```

A tool materializa cada receita sobre a série e injeta como `ind["alias"]`. A estratégia declara o que precisa, sem pré-materializar uma série.

## Quando usar inline vs pré-materializado

| Situação | Use |
|---|---|
| Backtest rápido, poucos indicadores | `indicadores_receitas` (inline) |
| Reutilizar indicadores em múltiplos backtests | Pipeline → `indicadores` (URI) |
| Indicadores complexos (pipeline multi-step) | Pipeline → `indicadores` (URI) |

---

> Próximo: [O simulador](./11-simulador.pt.md)
