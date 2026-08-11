# Recipe 10 — Full ML Pipeline

> **Level:** Advanced · **Premium**

End-to-end pipeline: features → lags → calendar → dataset → split → scaler → selection → model → CV → serving → backtest bridge.

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_full_ml",
    "nos": [
      { "id": "features", "componente": { "colunas": ["rsi", "atr", "momentum"] }, "entradas": ["ct://derived/btc_features"] },
      { "id": "lags", "componente": { "colunas": ["rsi", "atr", "momentum"], "lags": [1, 2, 3, 5, 10] }, "entradas": ["$features"] },
      { "id": "cal", "componente": { "campos": ["hora_dia", "dia_semana", "hora_sin", "hora_cos"] }, "entradas": ["$lags"] },
      { "id": "target", "componente": { "coluna": "close", "horizonte": 5 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": { "manter_features_nan": true }, "entradas": ["$cal", "$target"] },
      { "id": "impute", "componente": { "estrategia": "media" }, "entradas": ["$dataset"] },
      { "id": "split", "componente": { "k": 5, "embargo": 5 }, "entradas": ["$impute"] },
      { "id": "scaler", "componente": {}, "entradas": ["$split"] },
      { "id": "select", "componente": { "top_k": 15 }, "entradas": ["$scaler"] },
      { "id": "model", "componente": { "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 200, "max_depth": 5 } }, "entradas": ["$select"] },
      { "id": "evaluate", "componente": {}, "entradas": ["$model"] }
    ],
    "modelo": "$model",
    "predicao": "$model",
    "avaliacao": "$evaluate",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```
