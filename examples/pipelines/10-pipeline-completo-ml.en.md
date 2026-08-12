# Recipe 10 — Complete ML Pipeline

> **Level:** Advanced · **Requires:** Premium · **Prerequisites:** Recipe 2 (indicators), 6 (lags), 7 (dataset), 8 (walk-forward), 9 (model)

---

## The end-to-end pipeline

`montar_esteira_ml` lets you chain **all** machine-learning steps into a single DAG graph — from raw data (series + indicators) to final prediction and backtest evaluation. The 11 steps are:

1. **Fetch series** and generate indicators (RSI + ATR) via `materializar_indicador`
2. **Feature set** — select columns of interest
3. **Generate lags** — create lagged variables for time windows
4. **Apply to features** — merge lags with original feature set
5. **Calendar** — add seasonal fields (hour, day, sin/cos)
6. **Target direction** — define binary target (up/down) at a horizon
7. **Dataset** — cross features + target, align timestamps
8. **Impute** — fill NaN (lags create gaps at the start)
9. **Split walk-forward** — divide train/validation into `n_folds`
10. **Scaler + Selection** — normalize (z-score) and reduce dimensionality
11. **Model + Predict** — train GBDT and generate point predictions

---

## Step 1 — Fetch series and features

```python
# 1a. Fetch klines from Binance
buscar_binance(symbol="BTCUSDT", interval="15m", limit=2000)

# 1b. Materialize RSI and ATR on the series
materializar_indicador(
    serie="ct://series/binance/BTCUSDT/15m",
    indicador="rsi",
    parametros={"period": 14},
    destino="ct://derived/btc_rsi"
)
materializar_indicador(
    serie="ct://series/binance/BTCUSDT/15m",
    indicador="atr",
    parametros={"period": 14},
    destino="ct://derived/btc_atr"
)

# 1c. Compose single series with both features
compor_serie(
    series=["ct://derived/btc_rsi", "ct://derived/btc_atr"],
    destino="ct://derived/btc_ml_feats"
)
```

---

## Step 2 — Assemble complete pipeline

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_full_ml",
    "nos": [
      { "id": "feats", "componente": { "op": "feature_set", "colunas": ["rsi", "atr"] }, "entradas": ["ct://derived/btc_ml_feats"] },
      { "id": "lags", "componente": { "op": "gerar_lags", "colunas": ["rsi", "atr"], "lags": [1, 2, 3, 5, 10] }, "entradas": ["$feats"] },
      { "id": "feats_lag", "componente": { "op": "aplicar_em_features" }, "entradas": ["$lags", "$feats"] },
      { "id": "cal", "componente": { "op": "gerar_calendario", "campos": ["hora_dia", "dia_semana", "hora_sin", "hora_cos", "dia_sin", "dia_cos"] }, "entradas": ["$feats_lag"] },
      { "id": "alvo", "componente": { "op": "target_direcao", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "ds", "componente": { "op": "dataset", "manter_features_nan": true }, "entradas": ["$cal", "$alvo"] },
      { "id": "impute", "componente": { "op": "imputar", "estrategia": "media" }, "entradas": ["$ds"] },
      { "id": "wf", "componente": { "op": "split_walk_forward", "n_folds": 5, "rolling": false, "treino_inicial_frac": 0.5 }, "entradas": ["$impute"] },
      { "id": "scaler", "componente": { "op": "scaler_zscore" }, "entradas": ["$wf"] },
      { "id": "select", "componente": { "op": "selecionar_features", "top_k": 15 }, "entradas": ["$scaler"] },
      { "id": "modelo", "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 200, "max_depth": 5, "learning_rate": 0.05 } }, "entradas": ["$select", "$select"] },
      { "id": "pred", "componente": { "op": "prever" }, "entradas": ["$modelo", "$feats_lag"] }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$select",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

> **Note on the `modelo` node:** it receives `["$select", "$select"]` — the same feature-selection node provides both the train and validation splits (the DAG resolves the walk-forward internally).

---

## What each node does

| Node | `op` | Purpose |
|---|---|---|
| `feats` | `feature_set` | Selects columns (`rsi`, `atr`) from composed series |
| `lags` | `gerar_lags` | Creates lags [1, 2, 3, 5, 10] for each column |
| `feats_lag` | `aplicar_em_features` | Merges generated lags with original features |
| `cal` | `gerar_calendario` | Adds 6 seasonal fields (hour, day, sin/cos) |
| `alvo` | `target_direcao` | Defines binary target: went up (1) or down (0) in 5 bars |
| `ds` | `dataset` | Crosses features + target; `manter_features_nan` keeps lag NaNs |
| `impute` | `imputar` | Fills NaN with mean (`estrategia": "media"`) |
| `wf` | `split_walk_forward` | Splits into 5 walk-forward folds, initial training 50% |
| `scaler` | `scaler_zscore` | Normalizes features to mean 0, std 1 |
| `select` | `selecionar_features` | Keeps top 15 most relevant features |
| `modelo` | `modelo` | Trains GBDT (`n_estimators=200`, `max_depth=5`) |
| `pred` | `prever` | Generates point predictions from model + features |

---

## Step 3 — Backtest the prediction

The prediction (`$pred`) is automatically turned into a pseudo-indicator by the pipeline. To backtest manually:

```python
ct_backtest(
    serie="ct://series/binance/BTCUSDT/15m",
    indicador="ct://predictions/btc_full_ml",
    capital_inicial=1000,
    fee_pct=0.001
)
```

The `avalar_backtest` field in the pipeline config already triggers this evaluation automatically at the end of `montar_esteira_ml`.

---

## Why each preprocessing step matters

- **Impute** — Lags of window 10 produce 10 rows of NaN at the start. Without imputation, those rows are dropped and the dataset loses valuable training data.
- **Scaler z-score** — Linear models (logistic regression) and neural networks (MLP) require normalization. GBDT doesn't need it, but it doesn't hurt and keeps the pipeline generic.
- **Feature selection** — With lags of 2 columns × 5 windows + 6 calendar fields = 16+ features. Reducing to top 15 decreases overfitting and speeds up training.
- **Calendar** — Crypto markets have intraday patterns (e.g., higher volume at specific hours). Sin/cos encode cyclical patterns without exploding dimensionality.
- **Walk-forward** — Temporal validation prevents data leakage. `rolling=false` with `treino_inicial_frac=0.5` expands training set at each fold.

---

## Variations

- 🔁 **Switch model family:** replace `"familia": "gbdt"` with `"familia": "mlp"` and add layers in `hiperparametros` (`{"hidden_layers": [64, 32], "epochs": 50}`).
- 📐 **Add PCA:** insert a node `{ "op": "pca", "n_componentes": 10 }` between `scaler` and `select` to reduce dimensionality by variance.
- 🔀 **Vary n_folds:** increase to `n_folds=10` (more folds, smaller training per fold) or decrease to `3` (fewer folds, larger training, more robust per fold).
- 📈 **Regressor instead of classifier:** swap `"target_direcao"` for `"target_retorno"` and `"tarefa": "classificacao"` for `"tarefa": "regressao"` to predict return magnitude.
