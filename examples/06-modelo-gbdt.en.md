# 06 — GBDT Model: The Direction Learner

> **Level:** Advanced · **Premium** · **Prerequisites:** [Survival Test](./04-teste-sobrevivencia.en.md), [Cross-Asset](./05-cross-asset.en.md), [Rhai Strategy](../docs/05-backtest/03-estrategia-rhai.en.md)

In examples 04 and 05, you measured the **survival floor** — how much the adaptive manager loses without an entry criterion. In BTC, the floor is negative: EV of −0.095 réguas without fees and −0.92 with fees. The conclusion was clear: **your strategy needs an entry factor that adds at least +0.92 réguas of edge per trade** to overcome execution cost.

In this example, you'll build that entry factor using **Gradient Boosted Decision Trees (GBDT)** — an ML model that learns to predict the direction of the next candle from technical indicators. You will:

1. **Build features** with an indicator pipeline (RSI, MACD, ADX, Bollinger, ATR, lags).
2. **Train the GBDT model** via a declarative ML pipeline, with walk-forward validation.
3. **Apply the model** as a live indicator and run a backtest.
4. **Compare with the survival floor** and with buy-and-hold.
5. **Optimize hyperparameters** with grid search.
6. **Understand the limitations** — overfitting, regime change, and when the model stops working.

---

## The problem

You want to predict whether the next BTC candle will go **up** or **down**. With that prediction, your strategy buys or sells using the `grupo` lib's adaptive manager.

Without a model, direction is arbitrary — the survival test showed that bleeds. With a model that gets direction right, each hit becomes edge that overcomes execution cost. The question is: **can a GBDT learn direction from technical indicators?**

> **GBDT** (Gradient Boosted Decision Trees) is an ensemble of decision trees that iteratively corrects the errors of previous trees. It's the most widely used tabular ML model in quantitative finance — robust, non-linear, and effective with few features.

---

## Step 1 — Fetch data and build features

### Fetch the series

```
Fetch BTCUSDT at 15m from Binance.
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "interval": "15m" }
}
```

### Build the indicator pipeline

Features are the model's raw material. Let's materialize 8 indicators + price + volume into a single derived series:

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_gbdt_features",
    "output": "$features",
    "steps": [
      { "op": "rsi", "id": "rsi", "source": "$anchor", "column": "close", "period": 14 },
      { "op": "macd", "id": "macd", "source": "$anchor", "column": "close", "fast": 12, "slow": 26, "signal": 9 },
      { "op": "adx", "id": "adx", "source": "$anchor", "period": 14 },
      { "op": "bollinger", "id": "boll", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "sma", "id": "sma20", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "sma", "id": "sma50", "source": "$anchor", "column": "close", "period": 50 },
      { "op": "atr", "id": "atr", "source": "$anchor", "period": 14 },
      {
        "op": "compose",
        "id": "features",
        "columns": [
          { "as_column": "rsi",         "source": "$rsi",   "source_column": "rsi" },
          { "as_column": "macd",        "source": "$macd",  "source_column": "macd" },
          { "as_column": "macd_signal", "source": "$macd",  "source_column": "signal" },
          { "as_column": "adx",         "source": "$adx",   "source_column": "adx" },
          { "as_column": "plus_di",     "source": "$adx",   "source_column": "plus_di" },
          { "as_column": "minus_di",    "source": "$adx",   "source_column": "minus_di" },
          { "as_column": "atr",         "source": "$atr",   "source_column": "atr" },
          { "as_column": "volume",      "source": "$anchor","source_column": "volume" }
        ]
      }
    ]
  }
}
```

> The feature series becomes available at `ct://derived/btc_gbdt_features` with 1712 rows and 8 columns.

### Exploratory data analysis (EDA)

Before training, check the correlation of each feature with price (target):

```json
{
  "name": "analisar_dataset",
  "arguments": {
    "features": ["ct://derived/btc_gbdt_features"],
    "target": "ct://series/binance/BTCUSDT/15m",
    "target_coluna": "close",
    "quantis": [0.1, 0.25, 0.5, 0.75, 0.9]
  }
}
```

#### Feature × price correlation

| Feature | Correlation with `close` | Insight |
|---|---|---|
| `rsi` | +0.24 | Weak positive — RSI tracks price |
| `macd` | +0.37 | Moderate positive |
| `macd_signal` | +0.38 | Similar to MACD |
| `adx` | −0.16 | Weak negative — ADX rises in trends |
| `plus_di` | +0.45 | Moderate positive |
| `minus_di` | −0.36 | Moderate negative |
| `atr` | −0.12 | Weak negative |
| `volume` | −0.07 | Nearly zero |

> Correlation with price **is not what the model will predict**. The model predicts **direction** (up/down), not the absolute price. The correlation above shows `plus_di` and `macd` have linear relationships with price — but GBDT captures **non-linear** relationships that correlation misses.

---

## Step 2 — Train the GBDT model

The ML pipeline is a **declarative DAG** — each node is a component with inputs and outputs. The flow is:

```
feature_set → gerar_lags → aplicar_em_features → dataset → split_walk_forward → modelo → prever
                                ↑
                          target_direcao
```

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_gbdt_model",
    "nos": [
      {
        "id": "feats",
        "componente": { "op": "feature_set", "colunas": ["rsi", "macd", "macd_signal", "adx", "plus_di", "minus_di", "atr", "volume"] },
        "entradas": ["ct://derived/btc_gbdt_features"]
      },
      {
        "id": "lags",
        "componente": { "op": "gerar_lags", "colunas": ["rsi", "macd", "adx", "atr"], "lags": [1, 2, 3] },
        "entradas": ["$feats"]
      },
      {
        "id": "feats_lag",
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
        "componente": { "op": "dataset" },
        "entradas": ["$feats_lag", "$alvo"]
      },
      {
        "id": "wf",
        "componente": { "op": "split_walk_forward", "n_folds": 4, "rolling": false, "treino_inicial_frac": 0.5 },
        "entradas": ["$ds"]
      },
      {
        "id": "modelo",
        "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 100, "max_depth": 5, "learning_rate": 0.1, "random_state": 42 } },
        "entradas": ["$ds", "$wf"]
      },
      {
        "id": "pred",
        "componente": { "op": "prever" },
        "entradas": ["$modelo", "$feats_lag"]
      }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$wf"
  }
}
```

### What each node does

| Node | `op` | Purpose |
|---|---|---|
| `feats` | `feature_set` | Packs columns from the derived series as X matrix |
| `lags` | `gerar_lags` | Creates 12 extra features: `rsi_lag1`, `rsi_lag2`, `rsi_lag3`, `macd_lag1`, ... |
| `feats_lag` | `aplicar_em_features` | Reapplies lags to the feature_set (needed because lags → Ajuste, not FeatureSet) |
| `alvo` | `target_direcao` | Labels +1/0/-1: if next candle rose >0.1%, class +1; fell >0.1%, class -1; otherwise, 0 |
| `ds` | `dataset` | Joins features + target by timestamp; drops rows with NaN (lag warm-up) |
| `wf` | `split_walk_forward` | Splits into 4 temporal folds (expanding): training grows each fold |
| `modelo` | `modelo` | Trains GBDT: 100 trees, depth 5, learning rate 0.1 |
| `pred` | `prever` | Applies the model to the feature_set → prediction series (`ct://derived/btc_gbdt_model_pred`) |

### Training result

```json
{
  "modelo_uri": "ct://models/btc_gbdt_model",
  "predicao_uri": "ct://derived/btc_gbdt_model_pred",
  "familia": "gbdt",
  "classes": [-1, 0, 1],
  "colunas_x": [
    "rsi", "macd", "macd_signal", "adx", "plus_di", "minus_di", "atr", "volume",
    "rsi_lag1", "rsi_lag2", "rsi_lag3",
    "macd_lag1", "macd_lag2", "macd_lag3",
    "adx_lag1", "adx_lag2", "adx_lag3",
    "atr_lag1", "atr_lag2", "atr_lag3"
  ],
  "nos_executados": 8
}
```

20 features (8 base + 12 lags), 3 classes (up / flat / down). The model is persisted at `ct://models/btc_gbdt_model` — you can reuse it as many times as you want without retraining.

---

## Step 3 — Apply the model and run a backtest

### Apply (without retraining)

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "btc_gbdt_model",
    "fonte": "ct://derived/btc_gbdt_features",
    "nome": "btc_gbdt_pred"
  }
}
```

> Materializes the `pred` column at `ct://derived/btc_gbdt_pred` — the model as a live indicator.

### Backtest: model as signal

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_gbdt_pred",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "btc_gbdt_bt_fee"
  }
}
```

The strategy is simple: if the model predicts up (`pred > 0`), buy 1.0 BTC. If it predicts down (`pred < 0`), sell 1.0 BTC. Otherwise, stay flat.

### Result (1712 candles of BTCUSDT 15m, with 0.1% fee)

```json
{
  "uri": "ct://backtest/btc_gbdt_bt_fee",
  "num_trades": 374,
  "pnl_total": 12446.75,
  "pnl_bruto": 60441.83,
  "fees_totais": 47995.08,
  "retorno_total": 1.2447,
  "sharpe": 0.120,
  "sortino": 0.378,
  "win_rate": 0.393,
  "profit_factor": 2.186,
  "drawdown_max": 0.094,
  "num_wins": 147,
  "num_losses": 227,
  "avg_win": 156.07,
  "avg_loss": -46.23,
  "exposicao": 0.281
}
```

---

## Step 4 — Interpreting the results

### The GBDT overcomes fees

| Metric | Value | What it means |
|---|---|---|
| `pnl_total` | +$12,447 | **Net profit after fees** — the model generates real edge |
| `pnl_bruto` | +$60,442 | Without fees, the profit is $60k — the model gets direction right consistently |
| `fees_totais` | $47,995 | 374 trades × ~$128/trade — execution cost is high |
| `profit_factor` | 2.19 | Gains are 2.19× losses — the model has positive edge |
| `win_rate` | 39% | Less than half the trades win — but `avg_win` ($156) is 3.4× `avg_loss` ($46) |
| `exposicao` | 28% | The model only trades 28% of the time — stays flat when it lacks conviction |
| `drawdown_max` | 9.4% | Maximum drawdown is controlled |

### Comparison: GBDT vs survival floor vs buy-and-hold

| Strategy | `pnl_total` | `win_rate` | `profit_factor` | `sharpe` | `drawdown` |
|---|---|---|---|---|---|
| **GBDT (fee 0.1%)** | **+$12,447** | 39% | **2.19** | 0.12 | 9.4% |
| GBDT (no fee) | +$60,442 | 98% | 104.7 | 0.38 | 0.7% |
| Buy & Hold (fee 0.1%) | −$296 | — | 0 | 0.003 | 28.9% |
| Survival (no fee) | −$3,807 | 35% | — | — | — |
| Survival (fee 0.1%) | −$24,609 | 5% | — | — | — |
| Random (fee 0.1%) | −$110,912 | 8% | 0.07 | 0.025 | 11.1× |

### The insight

The GBDT **generates edge that overcomes execution cost**:
- Survival (arbitrary side) loses $24.6k with fees — the floor.
- GBDT gains $12.4k with fees — a **$37k difference** above the floor.
- Buy & Hold loses $296 in the period — the market had no clear trend.
- Random loses $110k — the worst case (high turnover, no edge).

> Without fees, the GBDT has a win rate of **98%** and profit factor of **104** — the model almost never gets direction wrong in the period. Fees eat 80% of the gross profit ($60k → $12k), but the net result is **positive**. This is the doctrine's goal: generate edge that overcomes execution cost.

### Why 98% win rate without fees?

The model predicts direction (up/down/flat). Most bars have very small movement (< 0.1%), which the model classifies as class 0 (flat). The strategy stays flat — no position, no gain, no loss. When the model predicts +1 or -1, it's usually right because the signal is strong enough to have conviction.

With fees, each trade costs ~$128 (0.1% of ~$64k), and 374 trades generate $48k in fees. The gross PnL of $60k covers the fees and still leaves $12k.

---

## Step 5 — Optimizing hyperparameters

The model was trained with `n_estimators=100, max_depth=5, learning_rate=0.1`. Would other values work better? `otimizar_hiperparametros` does grid search with temporal validation:

```json
{
  "name": "otimizar_hiperparametros",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "familia": "gbdt",
    "colunas": ["close", "volume"],
    "horizonte": 1,
    "limiar": 0.001,
    "estrategia": "grid",
    "grade": {
      "n_estimators": [50, 100, 200],
      "max_depth": [3, 5, 7]
    },
    "hiperparametros_base": { "learning_rate": 0.1, "random_state": 42 },
    "treino_frac": 0.7
  }
}
```

### Grid search results (9 combinations)

| `n_estimators` | `max_depth` | Accuracy |
|---|---|---|
| 50 | 3 | **70.6%** |
| 100 | 3 | **70.6%** |
| 200 | 3 | **70.6%** |
| 50 | 5 | 66.9% |
| 100 | 5 | 66.9% |
| 200 | 5 | 66.9% |
| 50 | 7 | 66.9% |
| 100 | 7 | 66.9% |
| 200 | 7 | 66.9% |

### The reading

- **`max_depth=3` is better than `5` or `7`**: shallow trees generalize better. Depth 7 overfits — memorizes training but loses in validation.
- **`n_estimators` doesn't matter with `max_depth=3`**: 50, 100, or 200 trees give the same accuracy. The model converges quickly with shallow trees.
- **70.6% accuracy** means the model gets direction right on 70% of bars. The remaining 30% is the error — which the adaptive manager (stop/take/trailing) tries to limit.

> The optimization confirms the doctrine: **simple models generalize better**. `max_depth=3` with 50 trees is enough. More complexity doesn't help — and can hurt out of sample.

---

## Anatomy of the ML pipeline

```
                    ┌─────────────────────────────────────────────────┐
                    │            ML PIPELINE (DAG)                     │
                    ├─────────────────────────────────────────────────┤
                    │                                                 │
                    │  ct://derived/btc_gbdt_features                 │
                    │  (RSI, MACD, ADX, Bollinger, ATR, volume)       │
                    │           │                                     │
                    │           ▼                                     │
                    │  ┌─────────────┐                                │
                    │  │ feature_set  │  8 features                   │
                    │  └──────┬──────┘                                │
                    │         │                                      │
                    │         ▼                                      │
                    │  ┌─────────────┐   ┌──────────────────┐          │
                    │  │  gerar_lags  │→ │ aplicar_em_feat   │ +12 lags │
                    │  └─────────────┘   └────────┬─────────┘          │
                    │                              │                   │
                    │  ct://series/...             │                   │
                    │  (BTCUSDT 15m)               │                   │
                    │       │                      │                   │
                    │       ▼                      ▼                   │
                    │  ┌──────────────┐    ┌──────────────┐           │
                    │  │target_direcao│    │   dataset     │           │
                    │  │  (+1/0/-1)   │→   │ (join + drop) │           │
                    │  └──────────────┘    └──────┬───────┘           │
                    │                              │                   │
                    │                              ▼                   │
                    │                    ┌──────────────────┐         │
                    │                    │ split_walk_fwd    │ 4 folds │
                    │                    │ (expanding 50%+)  │         │
                    │                    └────────┬─────────┘         │
                    │                              │                   │
                    │                              ▼                   │
                    │                    ┌──────────────────┐         │
                    │                    │  modelo (GBDT)   │         │
                    │                    │  100 trees, d=5  │         │
                    │                    └────────┬─────────┘         │
                    │                              │                   │
                    │                              ▼                   │
                    │                    ┌──────────────────┐         │
                    │                    │    prever        │         │
                    │                    │ ct://derived/    │         │
                    │                    │ btc_gbdt_pred    │         │
                    │                    └──────────────────┘         │
                    └─────────────────────────────────────────────────┘
```

### Pipeline components

| `op` | What it does | Inputs |
|---|---|---|
| `feature_set` | Packs columns as X matrix | series (URI) |
| `gerar_lags` | Creates lags x[t-1], x[t-2], ... as new features | `$feature_set` |
| `aplicar_em_features` | Reapplies an Ajuste (lags, scaler) to feature_set | `$ajuste`, `$feature_set` |
| `target_direcao` | Labels +1/0/-1 by sign of return N bars ahead | series (URI) |
| `dataset` | Joins features + target by timestamp; removes NaN | `$feature_set`, `$target` |
| `split_walk_forward` | Splits into K temporal folds (expanding or rolling) | `$dataset` |
| `split_holdout` | Splits into train (initial fraction) + validation | `$dataset` |
| `modelo` | Trains model (familia: gbdt, centroide, mlp; tarefa: classificacao, regressao) | `$dataset`, `$split` |
| `prever` | Applies trained model → prediction series | `$modelo`, `$feature_set` |
| `avaliar` | Evaluates prediction against dataset + split | `$pred`, `$dataset`, `$split` |
| `imputar` | Fills NaN (mean, median, zero, constant) | `$feature_set` |
| `scaler_zscore` | Standardizes features (mean 0, std 1) | `$feature_set` |
| `winsorize` | Caps outliers at 1%-99% quantiles | `$feature_set` |
| `selecionar_correlacao` | Keeps top-k features by |corr| with target | `$feature_set` |

---

## Limitations and caveats

### 1. Overfitting

The **98% win rate without fees** is suspicious. The model almost never gets direction wrong — but this may be because most bars have very small movement (`pred = 0`, "flat" class). The model gets class 0 right most of the time because it's the dominant class.

> To validate the model isn't just predicting the majority class, check **class balance** in the dataset. If 70% of bars are class 0, then 70% accuracy is the trivial baseline.

### 2. Look-ahead bias

`target_direcao` uses `close.shift(-1) / close - 1` — the return of the **next** bar. The model predicts the following bar using data up to the current bar. But note:

- `dataset` removes the last bar (NaN in target).
- `prever` applies the model to the entire feature_set, **including the last bar** — where the model doesn't have the real target.
- The backtest uses the prediction only on bars where `pred` exists.

### 3. Regime change

The model was trained on 1712 bars of BTCUSDT 15m. If the market regime changes (bull → bear, low vol → high vol), the model may stop working. Walk-forward validation mitigates this partially, but doesn't eliminate it.

> **Run the test periodically**. If accuracy drops below 55% (near random), it's time to retrain or review features.

### 4. Execution cost eats profits

Without fees, the profit is $60k. With 0.1% fees, it drops to $12k — **80% of profit goes to fees**. Reducing turnover (only trading when `|pred|` is large) can help:

```rhai
// Only trade when the model has strong conviction
if ind["pred"][0] > 0.5 { comprado(1.0) }
else if ind["pred"][0] < -0.5 { vendido(1.0) }
else { zerado() }
```

### 5. The model doesn't replace the manager

The GBDT predicts **direction**, not magnitude. It doesn't know if the move will be large or small. The adaptive manager (stop/take/trailing) is what limits the loss when the model is wrong and captures the gain when it's right. The two work **together**:

- **Model**: decides **when** to enter and **which side**
- **Manager**: decides **how** to exit (stop, take, trailing)

---

## Next steps

- **Combine model + `grupo` lib**: use the GBDT prediction as a trigger for the `grupo` lib — enter only when the model predicts direction, and let the manager handle the exit — see [Fork of the Doctrine](./11-fork-doutrina.en.md).
- **LSTM model**: train a recurrent neural network that captures long-range temporal dependency — see [LSTM Model](./07-modelo-lstm.en.md).
- **Optimize features**: add CT Lab indicators (ct_bop, ct_momento, ct_range) and check if they improve accuracy — see [CT Indicators](../docs/04-indicadores/README.en.md).

---

> Back to: [README](../README.en.md) · [Cross-Asset](./05-cross-asset.en.md) · [Survival Test](./04-teste-sobrevivencia.en.md)

_Last updated: 2026-08-11_
