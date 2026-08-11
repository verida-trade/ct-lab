# Recipe 06 — Train Direction Model (GBDT)

> **Level:** Advanced · **Premium** · **Prerequisites:** [ML](../docs/06-ml/)

Full pipeline: features → direction target → walk-forward split → GBDT → evaluate.

## Step 1 — Fetch series and create indicators

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_features",
    "receita": "#{\"rsi\": rsi(close, 14), \"atr\": atr(high, low, close, 14), \"momentum\": close - close[5]}"
  }
}
```

## Step 2 — Build ML pipeline

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_dir_5",
    "nos": [
      { "id": "features", "componente": { "colunas": ["rsi", "atr", "momentum"] }, "entradas": ["ct://derived/btc_features"] },
      { "id": "lags", "componente": { "colunas": ["rsi", "atr", "momentum"], "lags": [1, 2, 3] }, "entradas": ["$features"] },
      { "id": "target", "componente": { "coluna": "close", "horizonte": 5 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": {}, "entradas": ["$lags", "$target"] },
      { "id": "split", "componente": { "n_folds": 4, "treino_inicial_frac": 0.5 }, "entradas": ["$dataset"] },
      { "id": "scaler", "componente": {}, "entradas": ["$split"] },
      { "id": "model", "componente": { "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 100, "max_depth": 4, "learning_rate": 0.05 } }, "entradas": ["$scaler"] },
      { "id": "evaluate", "componente": {}, "entradas": ["$model"] }
    ],
    "modelo": "$model",
    "predicao": "$model",
    "avaliacao": "$evaluate"
  }
}
```

## Step 3 — Serving

```json
{
  "name": "aplicar_modelo",
  "arguments": { "modelo": "ct://models/btc_dir_5", "fonte": "ct://series/binance/BTCUSDT/15m", "probas": true }
}
```

## Step 4 — Backtest bridge

Add `avalar_backtest`:

```json
{ "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 } }
```

Default strategy: long if pred > 0, short if pred < 0.
