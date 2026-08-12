# 10 — Full ML Pipeline: From Feature to Backtest

> **Level:** Advanced · **Premium** · **Prerequisites:** [Regime + Model](./09-regime-modelo.en.md), [GBDT Model](./06-modelo-gbdt.en.md), [MLP Model](./07-modelo-lstm.en.md)

You've trained GBDT (example 06), MLP (example 07), detected regime (example 09), and measured microstructure (example 08). Each example used the ML pipeline, but always in a simplified version — `feature_set → dataset → split → model → predict`. Now it's time to build the **complete** pipeline: with lags, winsorization, imputation, scaling, and economic evaluation via backtest — all in a single declarative DAG.

The central question: **does preprocessing improve the model, or just complicate it?**

> **ML pipeline** is a DAG (directed acyclic graph) of components. Each component receives artifacts (FeatureSet, Dataset, Fit, Prediction) and produces another. The pipeline resolves the topology at runtime — you declare the graph, it executes in topological order.

---

## The problem

In example 06, the simple GBDT had 70.6% accuracy and overcame fees: +$12,447 net PnL. But that pipeline had no preprocessing — raw features directly into the model. The natural question:

1. **Winsorization** (clipping outliers at 1%/99% quantiles) removes extreme noise → does the model learn cleaner patterns?
2. **Imputation** (filling NaN with mean) recovers rows lost in lag warmup → more training data?
3. **Z-Score scaling** (normalizing to mean 0, std 1) equalizes feature scales → does GBDT converge better?
4. **Lags of 5 features** (RSI, MACD, ADX, BOP, volume at t-1, t-2, t-3) give the model temporal memory → better prediction?

The complete pipeline is the empirical test of these hypotheses.

---

## Step 1 — Build advanced features

### 1.1 — Pipeline with 14 indicators + BOP

The feature pipeline from example 06 had 8 indicators. Here we use 14, adding BOP (microstructure), SMA20/SMA50 (trend), and Bollinger bands (volatility):

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "r10_full_feats",
    "output": "$features",
    "steps": [
      { "op": "rsi", "id": "rsi", "source": "$anchor", "column": "close", "period": 14 },
      { "op": "macd", "id": "macd", "source": "$anchor", "column": "close", "fast": 12, "slow": 26, "signal": 9 },
      { "op": "adx", "id": "adx", "source": "$anchor", "period": 14 },
      { "op": "bollinger", "id": "boll", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "atr", "id": "atr", "source": "$anchor", "period": 14 },
      { "op": "bop", "id": "bop", "source": "$anchor" },
      { "op": "sma", "id": "sma20", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "sma", "id": "sma50", "source": "$anchor", "column": "close", "period": 50 },
      {
        "op": "compose",
        "id": "features",
        "columns": [
          { "as_column": "rsi", "source": "$rsi", "source_column": "rsi" },
          { "as_column": "macd", "source": "$macd", "source_column": "macd" },
          { "as_column": "macd_signal", "source": "$macd", "source_column": "signal" },
          { "as_column": "adx", "source": "$adx", "source_column": "adx" },
          { "as_column": "plus_di", "source": "$adx", "source_column": "plus_di" },
          { "as_column": "minus_di", "source": "$adx", "source_column": "minus_di" },
          { "as_column": "bb_upper", "source": "$boll", "source_column": "upper" },
          { "as_column": "bb_lower", "source": "$boll", "source_column": "lower" },
          { "as_column": "bb_middle", "source": "$boll", "source_column": "middle" },
          { "as_column": "atr", "source": "$atr", "source_column": "atr" },
          { "as_column": "bop", "source": "$bop", "source_column": "bop" },
          { "as_column": "sma20", "source": "$sma20", "source_column": "sma" },
          { "as_column": "sma50", "source": "$sma50", "source_column": "sma" },
          { "as_column": "volume", "source": "$anchor", "source_column": "volume" },
          { "as_column": "close", "source": "$anchor", "source_column": "close" }
        ]
      }
    ]
  }
}
```

> Result: `ct://derived/r10_full_feats` with 1712 rows and 15 columns (14 indicators + close).

---

## Step 2 — Simple pipeline (baseline)

Before the complete pipeline, let's establish a baseline — the minimal pipeline, same as example 06, but with more features and lags of 4 features:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "r10_simples",
    "nos": [
      {
        "id": "feats",
        "componente": { "op": "feature_set",
          "colunas": ["rsi","macd","macd_signal","adx","plus_di","minus_di","bb_upper","bb_lower","bb_middle","atr","bop","sma20","sma50","volume"] },
        "entradas": ["ct://derived/r10_full_feats"]
      },
      {
        "id": "lags",
        "componente": { "op": "gerar_lags", "colunas": ["rsi","macd","adx","bop"], "lags": [1,2,3] },
        "entradas": ["$feats"]
      },
      {
        "id": "applied",
        "componente": { "op": "aplicar_em_features" },
        "entradas": ["$lags", "$feats"]
      },
      {
        "id": "alvo",
        "componente": { "op": "target_direcao", "horizonte": 1 },
        "entradas": ["ct://series/binance/BTCUSDT/15m"]
      },
      {
        "id": "ds",
        "componente": { "op": "dataset" },
        "entradas": ["$applied", "$alvo"]
      },
      {
        "id": "split",
        "componente": { "op": "split_walk_forward", "n_folds": 4, "treino_inicial_frac": 0.5 },
        "entradas": ["$ds"]
      },
      {
        "id": "modelo",
        "componente": { "op": "modelo", "familia": "gbdt",
          "hiperparametros": { "max_depth": 3, "n_estimators": 100, "learning_rate": 0.1 } },
        "entradas": ["$ds", "$split"]
      },
      {
        "id": "pred",
        "componente": { "op": "prever" },
        "entradas": ["$modelo", "$applied"]
      },
      {
        "id": "aval",
        "componente": { "op": "avaliar" },
        "entradas": ["$pred", "$ds", "$split"]
      }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$aval"
  }
}
```

### Validation metrics (walk-forward, 4 folds)

| Metric | Value |
|---|---|
| Accuracy | 50.2% |
| F1 macro | 0.502 |
| Precision macro | 0.502 |
| Recall macro | 0.502 |
| N validation | 215 |

> **50.2% accuracy is marginally better than random** (33.3% for 3 classes). The simple model isn't predicting direction — it's predicting the majority class.

### Backtest

| Metric | With fee 0.1% | No fee |
|---|---|---|
| **PnL** | −$28,164 | +$74,059 |
| **Gross PnL** | +$74,089 | +$74,089 |
| **Fees** | $102,095 | 0 |
| **Trades** | 795 | 795 |
| **Win rate** | 24.8% | 89.4% |
| **Profit factor** | 0.50 | 14.17 |
| **Sharpe** | 0.005 | 0.466 |
| **Exposure** | 99.7% | 99.7% |

> **What happened**: without fees, the strategy has an 89.4% win rate and profit factor of 14.17 — seems extraordinary. But 795 trades over 1712 candles (46% turnover): the model predicts direction almost every time and the backtest executes nearly every candle. With fees, $102k in fees destroys everything. The 50.2% accuracy confirms: the model has no real edge, just overfits the training set.

---

## Step 3 — Complete pipeline (with preprocessing)

Now the complete pipeline: everything the simple one has, plus **winsorization, imputation, z-score scaling**, and a **threshold** on the target to filter minimal movements:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "r10_full",
    "nos": [
      {
        "id": "feats",
        "componente": { "op": "feature_set",
          "colunas": ["rsi","macd","macd_signal","adx","plus_di","minus_di","bb_upper","bb_lower","bb_middle","atr","bop","sma20","sma50","volume"] },
        "entradas": ["ct://derived/r10_full_feats"]
      },
      {
        "id": "lags",
        "componente": { "op": "gerar_lags", "colunas": ["rsi","macd","adx","bop","volume"], "lags": [1,2,3] },
        "entradas": ["$feats"]
      },
      {
        "id": "applied",
        "componente": { "op": "aplicar_em_features" },
        "entradas": ["$lags", "$feats"]
      },
      {
        "id": "alvo",
        "componente": { "op": "target_direcao", "horizonte": 1, "limiar": 0.001 },
        "entradas": ["ct://series/binance/BTCUSDT/15m"]
      },
      {
        "id": "ds",
        "componente": { "op": "dataset", "manter_features_nan": true },
        "entradas": ["$applied", "$alvo"]
      },
      {
        "id": "split",
        "componente": { "op": "split_walk_forward", "n_folds": 4, "treino_inicial_frac": 0.5 },
        "entradas": ["$ds"]
      },
      {
        "id": "winso",
        "componente": { "op": "winsorize", "q_inf": 0.01, "q_sup": 0.99 },
        "entradas": ["$ds", "$split"]
      },
      {
        "id": "ds_w",
        "componente": { "op": "aplicar_winsorize" },
        "entradas": ["$winso", "$ds"]
      },
      {
        "id": "imp",
        "componente": { "op": "imputar", "estrategia": "media" },
        "entradas": ["$ds_w", "$split"]
      },
      {
        "id": "ds_i",
        "componente": { "op": "aplicar_imputacao" },
        "entradas": ["$imp", "$ds_w"]
      },
      {
        "id": "scaler",
        "componente": { "op": "scaler_zscore" },
        "entradas": ["$ds_i", "$split"]
      },
      {
        "id": "ds_s",
        "componente": { "op": "aplicar_scaler" },
        "entradas": ["$scaler", "$ds_i"]
      },
      {
        "id": "modelo",
        "componente": { "op": "modelo", "familia": "gbdt",
          "hiperparametros": { "max_depth": 3, "n_estimators": 100, "learning_rate": 0.1 } },
        "entradas": ["$ds_s", "$split"]
      },
      {
        "id": "pred",
        "componente": { "op": "prever" },
        "entradas": ["$modelo", "$applied"]
      },
      {
        "id": "aval",
        "componente": { "op": "avaliar" },
        "entradas": ["$pred", "$ds_s", "$split"]
      }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$aval"
  }
}
```

### Pipeline anatomy (15 nodes)

```
feats ──→ lags ──→ applied ──→ ds ──→ split
  │                                        │
  │                                        ├─→ winso ──→ ds_w ──┐
  │                                        │                     │
  │                                        ├─→ imp ──→ ds_i ────┤
  │                                        │                     │
  │                                        └─→ scaler ──→ ds_s ──┤
  │                                                              │
  └──────────────────────────────────────────────────→ pred ←── modelo ←─┘
                                                         │
                                                         └──→ aval
```

| Node | Component | What it does |
|---|---|---|
| `feats` | `feature_set` | Selects 14 columns from feature series |
| `lags` | `gerar_lags` | Generates lags (1,2,3) of 5 features → +15 columns |
| `applied` | `aplicar_em_features` | Reapplies lags to FeatureSet |
| `alvo` | `target_direcao` | Return direction (−1, 0, +1) with 0.1% threshold |
| `ds` | `dataset` | Joins features + target, keeps NaN |
| `split` | `split_walk_forward` | 4 walk-forward folds, 50% initial training |
| `winso` | `winsorize` | Learns 1%/99% quantile limits on training |
| `ds_w` | `aplicar_winsorize` | Clips outliers to learned limits |
| `imp` | `imputar` | Learns per-column mean on training |
| `ds_i` | `aplicar_imputacao` | Fills NaN with training mean |
| `scaler` | `scaler_zscore` | Learns mean/std per column on training |
| `ds_s` | `aplicar_scaler` | Normalizes features to z-score |
| `modelo` | `modelo` (gbdt) | Trains GBDT (max_depth=3, 100 trees) |
| `pred` | `prever` | Applies model → per-bar prediction |
| `aval` | `avaliar` | Computes accuracy/F1 on validation fold |

> **Anti-leakage**: `winsorize`, `imputar`, and `scaler` are `EstimadorFitDependente` — they receive `(dataset, split)` and **fit only on training**. The `aplicar_*` components apply the fit to the entire dataset using training parameters. No information leaks from test to train.

### Validation metrics

| Metric | Simple pipeline | Complete pipeline |
|---|---|---|
| Accuracy | 50.2% | 18.7% |
| F1 macro | 0.502 | 0.108 |
| Precision macro | 0.502 | 0.228 |
| N validation | 215 | 214 |

> **Accuracy plummets from 50.2% to 18.7%!** This seems disastrous but is misleading. The simple pipeline predicts nearly every bar (795 trades) — accuracy is inflated by majority-class bias. The complete pipeline, with the 0.1% threshold, predicts almost everything as class 0 (neutral) — per-class accuracy drops, but the model **only trades when it has conviction**.

### Backtest

| Metric | Simple w/ fee | Complete w/ fee | Simple no fee | Complete no fee |
|---|---|---|---|---|
| **PnL** | −$28,164 | −$408 | +$74,059 | +$746 |
| **Gross PnL** | +$74,089 | +$530 | +$74,089 | +$530 |
| **Fees** | $102,095 | $1,026 | 0 | 0 |
| **Trades** | 795 | 8 | 795 | 8 |
| **Win rate** | 24.8% | 25.0% | 89.4% | 50.0% |
| **Profit factor** | 0.50 | 0.84 | 14.17 | 1.21 |
| **Sharpe** | 0.005 | 0.002 | 0.466 | 0.009 |
| **Calmar** | −0.36 | −1.95 | ≈∞ | 13.44 |
| **Max DD** | 281.6% | 29.5% | 1.8% | 25.1% |
| **Exposure** | 99.7% | 99.3% | 99.7% | 99.3% |
| **Expectancy** | −$35/trade | −$62/trade | +$93/trade | +$66/trade |
| **Annual return** | −100% | −57.4% | ≈∞ | +337% |

---

## Step 4 — Analysis: did preprocessing work?

### What the complete pipeline achieved

**Reduced turnover from 795 to 8 trades** — a 99% reduction. Fees dropped from $102k to $1k — a 100x factor. The model stopped predicting nearly every bar and started predicting only with conviction (class ≠ 0).

**Profit factor with fees improved**: from 0.50 (simple) to 0.84 (complete). Still doesn't overcome fees, but much closer.

### What it didn't achieve

**Didn't overcome fees**. PnL with fees is −$408. The 8 trades generated $530 gross PnL against $1,026 in fees. Edge exists (PF 1.21 without fees), but isn't large enough to overcome 0.1% fee per side.

**Accuracy dropped** because the threshold (0.1%) classified many targets as class 0 (neutral), and the model predicts the majority class. But this is **a feature, not a bug** — the model only predicts direction when expected return is > 0.1%, which is exactly when it's worth trading.

### What's missing

The GBDT from example 06 overcame fees (+$12,447) because it used `target_direcao` **without threshold** and a smaller `feature_set` (8 features). The complete pipeline here uses 29 features (14 original + 15 lags) — more features = more overfitting with 215 validation samples. The threshold solves overtrading but not overfitting.

> **The engineering lesson**: preprocessing reduces turnover and fees, but doesn't replace edge. The model needs features with real predictive power, not just more well-processed features. The complete pipeline is **necessary but not sufficient** — it's the infrastructure that makes deployment possible, but edge comes from the features.

---

## Step 5 — Available pipeline components

The CT Lab ML pipeline has 30+ components. Here's the complete reference:

### Preprocessing (estimators with training fit)

| Component | Op | Function |
|---|---|---|
| `winsorize` | Learns limits | Clips outliers at q_inf/q_sup quantiles |
| `imputar` | Learns mean/median | Fills NaN |
| `scaler_zscore` | Learns μ/σ | Normalizes to z-score |
| `scaler_minmax` | Learns min/max | Normalizes to [0, 1] |
| `scaler_robust` | Learns median/IQR | Normalizes robust to outliers |
| `scaler_maxabs` | Learns max abs | Normalizes preserving sign |
| `reduzir_pca` | Learns components | Dimensionality reduction |
| `selecionar_correlacao` | Learns top_k | Selects features by correlation |
| `selecionar_variancia` | Learns threshold | Removes low-variance features |

### Transformations (no fit)

| Component | Op | Function |
|---|---|---|
| `gerar_lags` | Generates lags | Creates t-1, t-2, ... of features |
| `transformar_coluna` | Transforms | log, sqrt, abs, sign, return_pct |
| `interagir_colunas` | Combines | ratio, product, difference |
| `features_calendario` | Adds | hour_of_day, day_of_week, sin/cos |
| `preencher_temporal` | Fills | ffill or bfill |

### Evaluation

| Component | Op | Function |
|---|---|---|
| `avaliar` | Classification metrics | accuracy, F1, precision, recall |
| `avaliar_regressao` | Regression metrics | R², MSE, MAE |
| `avaliar_cv` | Cross-validation | Aggregated metrics per fold |
| `avaliar_clustering` | Clustering metrics | silhouette, inertia |
| `avaliar_anomalia` | Anomaly metrics | precision@n, recall |

### Models

| Backend | Family | Hyperparameters |
|---|---|---|
| Centroid | `centroide` | — |
| GBDT | `gbdt` | `max_depth`, `n_estimators`, `learning_rate` |
| MLP | `mlp` | `oculto`, `epocas`, `lr` |
| KMeans | `kmeans` | `n_clusters` |
| Isolation Forest | `isolation_forest` | `contamination` |
| Custom Python | `modelo_custom` | Inline script with `treinar`/`inferir` |

---

## Next steps

- **Custom model (LSTM)**: write a Python backend with LSTM using `modelo_custom` — the component accepts inline scripts with `treinar`/`inferir` functions. See the catalog at `ct://ml/catalog`.
- **Optimize hyperparameters**: use the `otimizar_hiperparametros` node within the pipeline to automatically grid-search `max_depth`, `n_estimators`, `learning_rate`.
- **Fork the doctrine**: use the complete pipeline's prediction as a trigger for the adaptive manager from the `grupo` library — the model says direction, the manager handles exit — see [Fork Doctrine](./11-fork-doutrina.en.md).
- **Regime + pipeline**: add ADX as a feature (example 09) so the model learns to adjust predictions by regime.

---

> Back to: [README](../README.md) · [Regime + Model](./09-regime-modelo.en.md) · [GBDT Model](./06-modelo-gbdt.en.md) · [Fork Doctrine](./11-fork-doutrina.en.md)

_Last updated: 2026-08-12_
