# Receita 10 — Pipeline Completo de ML

> **Nível:** Avançado · **Premium**

Pipeline ponta-a-ponta: features → lags → calendário → dataset → split → scaler → seleção → modelo → CV → serving → ponte backtest.

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_full_ml",
    "nos": [
      { "id": "features", "componente": { "colunas": ["rsi", "atr", "momentum"] }, "entradas": ["ct://derived/btc_features"] },
      { "id": "lags", "componente": { "colunas": ["rsi", "atr", "momentum"], "lags": [1, 2, 3, 5, 10] }, "entradas": ["$features"] },
      { "id": "cal", "componente": { "campos": ["hora_dia", "dia_semana", "hora_sin", "hora_cos"] }, "entradas": ["$lags"] },
      { "id": "target", "componente": { "coluna": "close", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": { "manter_features_nan": true }, "entradas": ["$cal", "$target"] },
      { "id": "impute", "componente": { "estrategia": "media" }, "entradas": ["$dataset"] },
      { "id": "split", "componente": { "k": 5, "embargo": 5 }, "entradas": ["$impute"] },
      { "id": "scaler", "componente": {}, "entradas": ["$split"] },
      { "id": "select", "componente": { "top_k": 15 }, "entradas": ["$scaler"] },
      { "id": "modelo", "componente": { "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 200, "max_depth": 5, "learning_rate": 0.05 } }, "entradas": ["$select"] },
      { "id": "avaliar", "componente": {}, "entradas": ["$modelo"] }
    ],
    "modelo": "$modelo",
    "predicao": "$modelo",
    "avaliacao": "$avaliar",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

## O que este pipeline faz

1. Puxa RSI, ATR e momentum como features
2. Gera lags (1, 2, 3, 5, 10 barras) de cada feature
3. Adiciona features de calendário (hora, dia da semana, sen/cos cíclicos)
4. Cria target de direção (5 barras à frente)
5. Monta dataset mantendo NaNs para imputação
6. Imputa NaN com média
7. Split purged k-fold (5 folds, embargo 5 barras)
8. Normalização z-score
9. Seleção por correlação (top 15 features)
10. GBDT (200 árvores, depth 5)
11. Avaliação + backtest econômico
