# Comparação de Backtests

> Rode múltiplas variações lado a lado e compare métricas.

## `ct_comparar`

Compara uma estratégia com diferentes parâmetros em uma única chamada:

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

Isto roda 9 backtests (3×3 grid) e retorna a tabela comparativa.

## `ct_buscar_backtests`

Lista backtests anteriores:

```json
{ "name": "ct_buscar_backtests", "arguments": { "limit": 10 } }
```

Retorna os últimos 10 backtests com URI, nome, métricas resumidas.

---

> Próximo: [Indicadores inline](./10-indicadores-inline.pt.md)
