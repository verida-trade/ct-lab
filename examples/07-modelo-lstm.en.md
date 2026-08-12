# 07 — MLP Model: Neural Network with PyTorch

> **Level:** Advanced · **Premium** · **Prerequisites:** [GBDT Model](./06-modelo-gbdt.en.md), [Survival Test](./04-teste-sobrevivencia.en.md)

In [example 06](./06-modelo-gbdt.en.md), you trained a GBDT that overcame fees — +$12,447 net profit with 0.1% fee. But GBDT is a **tree-based** model: it partitions feature space into rectangular regions. Can a **neural network** — which can learn smooth non-linear relationships — do better?

In this example, you'll train an **MLP** (Multi-Layer Perceptron) via PyTorch, using the same features and the same ML pipeline as example 06. The goal is to compare:

- **GBDT** (trees) vs **MLP** (neural network)
- Which model generates more edge?
- Which is more robust to fees?
- How does each handle confidence and exposure?

> **MLP** is a feedforward neural network: input layer → hidden layer (with ReLU activation) → output layer (softmax for classification). It trains with gradient descent (Adam). CT Lab's `mlp` backend uses PyTorch and is built-in — no extra installation needed.

---

## The problem

You want to predict the direction of the next BTC candle, same as example 06. But instead of trees, you want to try a neural network. The motivation:

1. **Smooth non-linearity**: MLP learns smooth decision boundaries, not step functions.
2. **Calibrated confidence**: MLP produces probabilities (softmax) — you can filter trades by conviction.
3. **Less overfitting with small data**: MLP has fewer parameters than a 100-tree GBDT.

The question is: **does a simple MLP (32 neurons, 300 epochs) beat GBDT on the same dataset?**

> The `mlp` backend in CT Lab is embedded. It uses `torch==2.12.0` (installed automatically via `uv` in the Python environment). The model is `nn.Sequential(Linear → ReLU → Linear)` — the simplest neural network architecture.

---

## Step 1 — Reuse features from example 06

Example 06 already built the feature pipeline at `ct://derived/btc_gbdt_features` (RSI, MACD, ADX, Bollinger, ATR, volume). The MLP will use **exactly the same features** — the comparison is fair.

```
Features are already at ct://derived/btc_gbdt_features (8 columns, 1712 rows).
```

If you need to rebuild, run the pipeline from [example 06, Step 1](./06-modelo-gbdt.en.md).

---

## Step 2 — Train the MLP model

The pipeline is identical to GBDT — only the `familia` in the `modelo` node changes:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_mlp_model",
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
        "componente": {
          "op": "modelo",
          "familia": "mlp",
          "tarefa": "classificacao",
          "hiperparametros": { "oculto": 32, "epocas": 300, "lr": 0.01 }
        },
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

### What changes compared to GBDT

| Field | GBDT | MLP |
|---|---|---|
| `familia` | `"gbdt"` | `"mlp"` |
| `hiperparametros` | `n_estimators`, `max_depth`, `learning_rate` | `oculto`, `epocas`, `lr` |
| Backend | scikit-learn | PyTorch |
| Dependency | `scikit-learn==1.9.0` | `torch==2.12.0` |

### Training result

```json
{
  "modelo_uri": "ct://models/btc_mlp_model",
  "predicao_uri": "ct://derived/btc_mlp_model_pred",
  "familia": "mlp",
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

Same 20 features, same 3 classes. The difference is inside the model: the MLP has 32 neurons in the hidden layer (32×20 + 32×3 = 736 parameters), while GBDT has 100 trees of depth 5.

---

## Step 3 — Apply the model and run a backtest

### Apply

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "btc_mlp_model",
    "fonte": "ct://derived/btc_gbdt_features",
    "nome": "btc_mlp_pred"
  }
}
```

### Backtest (with fees)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_mlp_pred",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "btc_mlp_bt_fee"
  }
}
```

### Result (with 0.1% fee)

```json
{
  "num_trades": 56,
  "pnl_total": -4056.30,
  "pnl_bruto": 3122.70,
  "fees_totais": 7179.00,
  "retorno_total": -0.406,
  "sharpe": -0.059,
  "sortino": -0.070,
  "win_rate": 0.268,
  "profit_factor": 0.424,
  "drawdown_max": 0.407,
  "num_wins": 15,
  "num_losses": 41,
  "avg_win": 199.0,
  "avg_loss": -171.7,
  "exposicao": 0.054
}
```

### Result (no fee)

```json
{
  "num_trades": 56,
  "pnl_total": 3122.70,
  "pnl_bruto": 3122.70,
  "fees_totais": 0,
  "retorno_total": 0.312,
  "sharpe": 0.047,
  "sortino": 0.080,
  "win_rate": 0.607,
  "profit_factor": 1.983,
  "drawdown_max": 0.066,
  "exposicao": 0.054
}
```

---

## Step 4 — Comparison: MLP vs GBDT

### Comparative table (same features, same lags, same walk-forward)

| Metric | MLP (fee 0.1%) | GBDT (fee 0.1%) | MLP (no fee) | GBDT (no fee) |
|---|---|---|---|---|
| `pnl_total` | **−$4,056** | **+$12,447** | +$3,123 | +$60,442 |
| `pnl_bruto` | $3,123 | $60,442 | $3,123 | $60,442 |
| `num_trades` | 56 | 374 | 56 | 374 |
| `win_rate` | 27% | 39% | 61% | 98% |
| `profit_factor` | 0.42 | 2.19 | 1.98 | 104.7 |
| `exposicao` | 5.4% | 28.1% | 5.4% | 28.1% |
| `drawdown_max` | 40.7% | 9.4% | 6.6% | 0.7% |
| `sharpe` | −0.059 | 0.120 | 0.047 | 0.384 |

### The reading

1. **GBDT overcomes fees; MLP doesn't.** GBDT generates $60k of gross PnL — $48k in fees still leaves $12k profit. MLP generates only $3.1k of gross PnL — the $7.2k in fees eats it all and still leaves −$4k.

2. **MLP is much more selective.** 56 trades vs 374 — exposure 5.4% vs 28.1%. The MLP only trades when it has conviction (class +1 or -1), and stays flat on most bars (class 0). GBDT trades much more because it classifies more bars as +1 or -1.

3. **Fewer trades = less gross edge.** With only 56 trades, MLP accumulates $3.1k of gross PnL. Even with lower fees ($7.2k vs GBDT's $48k), the gross PnL is so low it doesn't cover even the reduced fees.

4. **Win rate without fees: MLP 61% vs GBDT 98%.** GBDT almost never gets direction wrong when it predicts +1 or -1. MLP gets it wrong 39% of the time. GBDT's advantage is directional robustness — it iteratively prunes errors.

> The lesson isn't "MLP is worse than GBDT." It's that **with this dataset and this configuration**, GBDT extracts more edge. MLP is a simpler model (736 parameters vs hundreds of tree nodes) — it might be that with more data, feature normalization, or a richer architecture (LSTM, Transformer), the neural network would win. But with 1712 bars and 20 features, GBDT wins.

---

## Step 5 — Confidence filter with probabilities

The MLP produces probabilities via softmax. Instead of always trading when `pred ≠ 0`, we can filter by conviction:

### Apply with probabilities

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "btc_mlp_model",
    "fonte": "ct://derived/btc_gbdt_features",
    "nome": "btc_mlp_probas",
    "probas": true
  }
}
```

> Materializes 3 columns: `p_0` (probability of flat), `p_1` (up), `p_m1` (down).

### Backtest with confidence filter

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_mlp_probas",
    "estrategia_script": "if ind[\"p_1\"][0] > 0.6 { comprado(1.0) } else if ind[\"p_m1\"][0] > 0.6 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "btc_mlp_confidence"
  }
}
```

Only trades when the probability of a class exceeds 60% — much more conservative.

### Result (60% filter)

```json
{
  "num_trades": 9,
  "pnl_total": -803.09,
  "pnl_bruto": 353.25,
  "fees_totais": 1156.34,
  "win_rate": 0.333,
  "profit_factor": 0.253,
  "exposicao": 0.005
}
```

### The reading

The confidence filter reduces from 56 to **9 trades** — but the result worsens: PnL drops from −$4,056 to −$803, but gross PnL drops more ($3,123 → $353). The filter removes good and bad trades in similar proportion — the MLP's softmax confidence isn't discriminative enough.

> The confidence filter works better when the model is **well calibrated** (probability = actual frequency). Neural networks are often poorly calibrated — they produce overconfident or underconfident distributions. GBDT, on the other hand, doesn't produce reliable probabilities by default (sklearn's "probabilities" are smoothed).

---

## Step 6 — Anatomy of the MLP backend

The `mlp` backend is embedded in the MCP server and runs via PyTorch in the `uv` environment. The architecture is:

```
                    ┌────────────────────────────────────┐
                    │        MLP Backend (PyTorch)       │
                    ├────────────────────────────────────┤
                    │                                    │
                    │  Input: X (N x 20)               │
                    │           │                        │
                    │           ▼                        │
                    │  ┌──────────────┐                  │
                    │  │ nn.Linear(20,│  weights + bias  │
                    │  │       32)    │  (640 + 32)     │
                    │  └──────┬───────┘                  │
                    │         │                          │
                    │         ▼                          │
                    │  ┌──────────────┐                  │
                    │  │  nn.ReLU()   │  activation      │
                    │  └──────┬───────┘                  │
                    │         │                          │
                    │         ▼                          │
                    │  ┌──────────────┐                  │
                    │  │ nn.Linear(32,│  weights + bias  │
                    │  │        3)    │  (96 + 3)      │
                    │  └──────┬───────┘                  │
                    │         │                          │
                    │         ▼                          │
                    │  CrossEntropyLoss (classification)  │
                    │  Adam optimizer (lr=0.01)          │
                    │  300 epochs                        │
                    │                                    │
                    │  Total: 739 parameters             │
                    │  (vs GBDT: 100 trees × ~32 nodes)  │
                    └────────────────────────────────────┘
```

### MLP hyperparameters

| Parameter | Default | Effect |
|---|---|---|
| `oculto` | 16 | Number of hidden neurons. More = more capacity, more overfitting |
| `epocas` | 200 | Number of gradient descent steps. More = better training fit, overfit risk |
| `lr` | 0.01 | Adam learning rate. High = fast convergence, oscillation risk |
| `random_state` | — | torch seed (not controlled by MLP backend) |

> To tune these hyperparameters, use `otimizar_hiperparametros` as in example 06 — just change `familia` to `"mlp"`.

---

## GBDT vs MLP: when to use each?

| Criterion | GBDT | MLP |
|---|---|---|
| **Small dataset** (< 5k) | ✓ Excellent | ✗ Overfit risk |
| **Large dataset** (> 50k) | ✓ Good (but slow) | ✓ Good |
| **Categorical features** | ✓ Native | ✗ Needs embedding |
| **Smooth relationships** | ✗ Step boundaries | ✓ Smooth boundaries |
| **Interpretability** | ✓ Feature importance | ✗ Black box |
| **Calibration** | ✗ Not calibrated | ✗ Not calibrated (better than GBDT) |
| **Fast training** | ✓ Seconds | ✓ Seconds (simple MLP) |
| **Temporal sequence** | ✗ No memory | ✗ No memory |
| **Turnover** | High (many trades) | Low (selective) |

> **Neither GBDT nor MLP has temporal memory.** Both see one bar at a time, without context of previous bars. Lags (lag1, lag2, lag3) are a "patch" — they give the model a window, but limited. An **LSTM** (Long Short-Term Memory) has internal memory and can capture long-range dependencies — it's the natural next step.

---

## Next steps

- **Combine MLP + adaptive manager**: use the MLP prediction as a trigger for the `grupo` lib — enter only when the model predicts direction, let the manager handle the exit — see [Fork of the Doctrine](./11-fork-doutrina.en.md).
- **Custom model (LSTM)**: write a Python backend with LSTM using `modelo_custom` — see [Full ML Pipeline](./10-esteira-ml-completa.en.md).
- **Optimize MLP hyperparameters**: run grid search varying `oculto`, `epocas`, `lr` — see [GBDT Model, Step 5](./06-modelo-gbdt.en.md).

---

> Back to: [README](../README.en.md) · [GBDT Model](./06-modelo-gbdt.en.md) · [Survival Test](./04-teste-sobrevivencia.en.md)

_Last updated: 2026-08-12_
