# Recipe 06 — Train a Direction Model (GBDT)

**Level:** Advanced · **Plan:** Premium · **Prerequisites:** Recipe 01 (Series), Recipe 02 (Backtest), Recipe 04 (Indicators)

This recipe builds a complete machine-learning pipeline using Gradient Boosted Decision Trees (GBDT) to predict price direction over 5-bar windows, with a backtest of the prediction.

---

## The ML pipeline

The CT Lab flow for predictive modeling is linear and declarative:

- **Feature set** → selects indicator columns to use as model features
- **Lags** → generates lagged versions (t-1, t-2, t-3) to give the model memory
- **Target** → creates the direction target (up/down/neutral) at a defined horizon
- **Split** → divides the dataset into walk-forward folds for out-of-sample validation
- **Model** → trains the GBDT with configurable hyperparameters
- **Predict** → generates predictions usable as an indicator in backtest

---

## Step 1 — Fetch series

```
buscar_binance(symbol="BTCUSDT", interval="15m")
```

Result: 1,724 candles cached at `ct://series/binance/BTCUSDT/15m`.

---

## Step 2 — Materialize features

We use `materializar_indicador` to create indicators serving as features. The recipe is a Rhai map with valid expressions:

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_ml_feats",
    "receita": "#{\"rsi\": rsi(close, 14), \"atr\": atr(high, low, close, 14)}"
  }
}
```

**Result:**

| Field        | Value                          |
|--------------|--------------------------------|
| uri          | `ct://derived/btc_ml_feats`    |
| row_count    | 1,724                          |
| value_names  | `["atr", "rsi"]`              |

> **Important:** Use supported Rhai functions (`rsi`, `atr`, `adx`, `macd`). For `macd`, pass all 3 periods: `macd(close, 12, 26, 9)["hist"]`. Avoid indexers like `close[5]` (unavailable). For `adx`, multiple columns are returned.

---

## Step 3 — Assemble ML pipeline

The `montar_esteira_ml` function takes an array of nodes, each with `id`, `componente` (containing an `op` field), and `entradas`:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_gbdt_dir5",
    "nos": [
      { "id": "feats",      "componente": { "op": "feature_set", "colunas": ["rsi", "atr"] }, "entradas": ["ct://derived/btc_ml_feats"] },
      { "id": "lags",       "componente": { "op": "gerar_lags", "colunas": ["rsi", "atr"], "lags": [1, 2, 3] }, "entradas": ["$feats"] },
      { "id": "feats_lag",  "componente": { "op": "aplicar_em_features" }, "entradas": ["$lags", "$feats"] },
      { "id": "alvo",       "componente": { "op": "target_direcao", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "ds",         "componente": { "op": "dataset" }, "entradas": ["$feats_lag", "$alvo"] },
      { "id": "wf",         "componente": { "op": "split_walk_forward", "n_folds": 4, "rolling": false, "treino_inicial_frac": 0.5 }, "entradas": ["$ds"] },
      { "id": "modelo",    "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 100, "max_depth": 4, "learning_rate": 0.05 } }, "entradas": ["$ds", "$wf"] },
      { "id": "pred",      "componente": { "op": "prever" }, "entradas": ["$modelo", "$feats_lag"] }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$wf",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

**Result:**

| Field           | Value                                                        |
|-----------------|--------------------------------------------------------------|
| modelo_uri      | `ct://models/btc_gbdt_dir5`                                  |
| predicao_uri    | `ct://derived/btc_gbdt_dir5_pred`                            |
| familia         | `gbdt`                                                       |
| colunas_x       | `rsi, atr, rsi_lag1, rsi_lag2, rsi_lag3, atr_lag1, atr_lag2, atr_lag3` |
| classes         | `[-1, 0, 1]`                                                 |
| nos_executados  | 8                                                            |

---

## What each node does

| Node        | op                   | Purpose                                                            |
|-------------|----------------------|--------------------------------------------------------------------|
| `feats`     | `feature_set`        | Selects columns (`rsi`, `atr`) from the materialized indicator     |
| `lags`      | `gerar_lags`         | Creates 1, 2, and 3-bar lags for each feature                     |
| `feats_lag` | `aplicar_em_features`| Combines lags with original features into a unified set           |
| `alvo`      | `target_direcao`     | Generates direction target (horizon=5, threshold=0.0): +1, 0 or -1|
| `ds`        | `dataset`            | Assembles the dataset (X = features + lags, y = target)           |
| `wf`        | `split_walk_forward` | Splits into 4 walk-forward folds, initial training fraction 50%    |
| `modelo`    | `modelo`             | Trains GBDT (100 trees, depth 4, LR 0.05)                          |
| `pred`      | `prever`             | Generates predictions using the trained model on features         |

---

## Step 4 — Backtest the prediction

The prediction URI contains a `pred` column (-1, 0, or 1). We use `ct_backtest` with the prediction as an indicator:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_gbdt_dir5_pred",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "gbdt_bt_fee"
  }
}
```

---

## Real results

| Metric            | With fee (0.1%)     | No fee               |
|-------------------|---------------------|----------------------|
| Trades            | 435                 | 435                  |
| Net PnL           | **-$29,999.25**     | **+$25,971.95**      |
| Gross PnL         | +$25,932.87         | +$25,932.87          |
| Total fees        | $55,843.66          | —                    |
| Sharpe            | 0.024               | 0.142                |
| Win rate          | 20.7%               | 69.4%                |
| Profit factor     | 0.337               | 3.357                |
| Max drawdown      | 30.0%               | —                    |
| Exposure          | 99.7%               | 99.7%                |
| Avg win           | +$168.71            | +$168.71             |
| Avg loss          | -$130.71            | -$130.71             |
| Payoff            | 1.291               | 1.291                |
| Long / Short      | 218 / 217           | 218 / 217            |

---

## The ML fee dilemma

The results reveal a classic ML-in-trading problem:

- **Massive gross edge:** +$25.9K gross PnL, 69.4% win rate without fees, PF = 3.357 — the model is extraordinarily predictive.
- **Fee trap:** 435 trades generate $55.8K in fees — **2× larger than the edge**. Net PnL sinks to -$30K.
- **Root cause:** 99.7% exposure means the model trades on virtually every bar. It predicts direction correctly but doesn't filter noise — it churns capital.
- **The core problem:** the model uses discrete classes (-1, 0, 1) with no confidence threshold. Every prediction becomes a trade, even when the model is uncertain.

### Suggested solutions

1. **Prediction threshold:** only trade if `|pred|` or the probability exceeds a minimum level (e.g., `prob > 0.6`).
2. **Use probabilities:** instead of hard class labels, access `predict_proba` and trade only above a minimum confidence.
3. **Increase horizon (`horizonte`):** larger horizons naturally reduce trade frequency.
4. **Additional filter:** use ATR or volatility to avoid trading in sideways market conditions.

---

## Variations

- **Add features:** include `ema`, `bollinger`, or `stochastic` in the `materializar_indicador` recipe to enrich the signal.
- **Increase horizon:** test `horizonte: 10` or `horizonte: 15` to reduce trade frequency and optionally capture larger moves.
- **Add scaler:** insert a `scaler_zscore` node before the `dataset` to normalize features (useful for linear models; GBDT is scale-invariant but it doesn't hurt).
- **Use probabilities:** replace `prever` with a node that extracts `predict_proba` and apply a confidence threshold in the backtest script.
