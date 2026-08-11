# Recipe 09 — Regime + Predictive Model

> **Level:** Advanced · **Premium** · **Prerequisites:** [ML](../docs/06-ml/), [Premium indicators](../docs/04-indicadores/02-indicadores-premium.en.md)

Use `ct_tendencia` as regime feature in an ML model.

## Step 1 — Fetch series and create ct_tendencia

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{
  "name": "ct_tendencia",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "janela_zscore": 96,
    "janela_atr": 14,
    "k_mov": 3,
    "z_min": 2.0,
    "tau_r": 1.5
  }
}
```

`ct_tendencia` returns 5 columns: `tend_ativa`, `direcao`, `fase`, `progresso`, `nivel_rompido`.

## Step 2 — Build ML pipeline with regime as feature

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_regime_model",
    "nos": [
      { "id": "features", "componente": { "colunas": ["direcao", "fase", "progresso"] }, "entradas": ["ct://derived/ct_tendencia_..."] },
      { "id": "lags", "componente": { "colunas": ["direcao", "fase", "progresso"], "lags": [1, 2, 3, 5] }, "entradas": ["$features"] },
      { "id": "target", "componente": { "coluna": "close", "horizonte": 5 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": {}, "entradas": ["$lags", "$target"] },
      { "id": "split", "componente": { "n_folds": 4, "treino_inicial_frac": 0.5 }, "entradas": ["$dataset"] },
      { "id": "scaler", "componente": {}, "entradas": ["$split"] },
      { "id": "model", "componente": { "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 150, "max_depth": 5 } }, "entradas": ["$scaler"] },
      { "id": "evaluate", "componente": {}, "entradas": ["$model"] }
    ],
    "modelo": "$model",
    "predicao": "$model",
    "avaliacao": "$evaluate",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

> **Note:** Regime is SETUP, not foundation. The model must beat the arbitrary-side floor in the same test.
