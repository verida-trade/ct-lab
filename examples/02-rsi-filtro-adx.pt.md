# Receita 02 — RSI com Filtro ADX

> **Nível:** Intermediário · Pipeline: RSI + ADX → sinal condicional → backtest

Esta receita constrói um sinal que só opera quando a tendência é forte (ADX > 25) e o RSI está em oversold/overbought.

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
      { "id": "rsi_low", "operacao": "condicional", "condicao": "$rsi", "coluna_condicao": "rsi", "entao": {"escalar": 0.0}, "senao": {"escalar": 1.0}, "coluna_saida": "oversold" },
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

> [Receita 03 — Backtest com lib grupo](./03-lib-grupo-backtest.pt.md)
