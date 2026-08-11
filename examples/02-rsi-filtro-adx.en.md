# Recipe 02 — RSI with ADX Filter

> **Level:** Intermediate · Pipeline: RSI + ADX → conditional signal → backtest

## Pipeline

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "rsi_adx_signal",
    "output": "$signal",
    "steps": [
      { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
      { "id": "adx", "operacao": "adx", "source": "$anchor" },
      { "id": "adx_strong", "operacao": "comparar", "esquerda": "$adx", "coluna_esquerda": "adx", "direita": {"escalar": 25.0}, "operador": "maior" },
      { "id": "signal", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$rsi_low"},{"fonte":"$adx_strong"}], "coluna_saida": "signal" }
    ]
  }
}
```

## Backtest

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" },
      "adx": { "receita": "adx(high, low, close)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_adx"
  }
}
```

---

> [Recipe 03 — Backtest with grupo lib](./03-lib-grupo-backtest.en.md)
