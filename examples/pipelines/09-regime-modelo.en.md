# Recipe 09 — Regime + Predictive Model

> **Level:** Advanced · **Premium:** Yes · **Time:** ~15 min

**Prerequisites:** Recipes 06 (ML pipeline) and 08 (backtests and survival test).

---

## What is regime?

Regime is the market state at a given moment — active trend, direction, cycle
phase, and progress. CT Lab detects regime via `ct_tendencia`, which transforms
raw OHLCV into five columns:

- **`tend_ativa`** — 1 if a statistically significant trend is active, 0 otherwise
- **`direcao`** — +1 (up), −1 (down), or 0 (sideways)
- **`fase`** — cycle stage (formation / confirmation / exhaustion)
- **`progresso`** — howfar along the trend is (0 to 1)
- **`nivel_rompido`** — breakout price that validated the trend

> **Key concept:** Regime is *setup*, not foundation. The model must beat the
> random-side floor in the same survival test.

---

## Step 1 — Fetch series and create ct_tendencia

```python
# 1-A: fetch 15m candles for BTCUSDT
buscar_binance(symbol="BTCUSDT", interval="15m", limit=2000)
#  → ct://series/binance/BTCUSDT/15m

# 1-B: detect regime with validated parameters
ct_tendencia(
    uri="ct://series/binance/BTCUSDT/15m",
    janela_zscore=96,
    janela_atr=14,
    k_mov=3,
    z_min=2.0,
    tau_r=1.5,
)
#  → ct://derived/ct_tendencia_binance_BTCUSDT_15m
#    columns: tend_ativa, direcao, fase, progresso, nivel_rompido
```

| Parameter       | Value | Purpose                                    |
|-----------------|-------|--------------------------------------------|
| `janela_zscore` | 96    | Window for z-score normalization           |
| `janela_atr`    | 14    | ATR window to normalize price moves        |
| `k_mov`         | 3     | Minimum movement multiplier                |
| `z_min`         | 2.0   | Minimum z-score to confirm a trend         |
| `tau_r`         | 1.5   | Minimum persistence (in bars)              |

---

## Step 2 — Materialize RSI features

```python
materializar_indicador(
    uri="ct://series/binance/BTCUSDT/15m",
    indicador="rsi",
    periodo=14,
    destino="ct://derived/btc_rsi_14",
)
#  → ct://derived/btc_rsi_14  (column: rsi)
```

---

## Step 3 — Build ML pipeline

The pipeline combines regime features (`direcao`, `fase`, `progresso`) with RSI
and generates lags to give the model temporal context.

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_regime_model",
    "nos": [
      { "id": "feats", "componente": { "op": "feature_set", "colunas": ["direcao", "fase", "progresso", "rsi"] }, "entradas": ["ct://derived/ct_tendencia_binance_BTCUSDT_15m", "ct://derived/btc_rsi_14"] },
      { "id": "lags", "componente": { "op": "gerar_lags", "colunas": ["direcao", "fase", "progresso", "rsi"], "lags": [1, 2, 3, 5] }, "entradas": ["$feats"] },
      { "id": "feats_lag", "componente": { "op": "aplicar_em_features" }, "entradas": ["$lags", "$feats"] },
      { "id": "alvo", "componente": { "op": "target_direcao", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "ds", "componente": { "op": "dataset" }, "entradas": ["$feats_lag", "$alvo"] },
      { "id": "wf", "componente": { "op": "split_walk_forward", "n_folds": 4, "rolling": false, "treino_inicial_frac": 0.5 }, "entradas": ["$ds"] },
      { "id": "modelo", "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 150, "max_depth": 5 } }, "entradas": ["$ds", "$wf"] },
      { "id": "pred", "componente": { "op": "prever" }, "entradas": ["$modelo", "$feats_lag"] }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$wf",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

Result: `ct://derived/btc_regime_model_pred`

### What each node does

| Node        | Op                   | Purpose                                                           |
|-------------|----------------------|-------------------------------------------------------------------|
| `feats`     | `feature_set`        | Selects regime + RSI columns from source URIs                     |
| `lags`      | `gerar_lags`         | Creates lag features (1, 2, 3, 5) for 4 columns → temporal context |
| `feats_lag` | `aplicar_em_features`| Merges original features with lags into a single DataFrame         |
| `alvo`      | `target_direcao`     | Creates binary target: price direction in 5 bars (up/down)        |
| `ds`        | `dataset`            | Assembles dataset (X = feats+lags, y = target)                    |
| `wf`        | `split_walk_forward` | Splits into 4 walk-forward folds (50% initial training fraction)  |
| `modelo`    | `modelo`             | Trains GBDT (150 trees, depth 5) on each fold                     |
| `pred`      | `prever`             | Generates point-in-time predictions using the trained model       |

---

## Step 4 — Backtest the prediction

The model's prediction is materialized as a derived series and fed into the
backtest as a signal indicator.

```python
ct_backtest(
    uri="ct://derived/btc_regime_model_pred",
    capital_inicial=1000,
    fee_pct=0.001,
    survival=True,
)
```

Compare the result against the **random-side floor** (Recipe 08) over the same
period. If the model does not beat the floor, regime features alone are not
enough — go back to Steps 1-3.

---

## Interpretation

- **Regime features** give the model context: active trend, direction, and
  phase help the GBDT distinguish momentum moments from sideways markets.
- **Lags** capture dynamics: the regime 3 bars ago matters as much as the
  current regime when predicting the next direction.
- **Walk-forward** avoids look-ahead bias — each fold trains only on data
  prior to the test window.
- **The model must beat the survival floor.** Regime improves the signal, but
  does not guarantee a statistical edge if the series is too efficient.
- **Regime is setup, not foundation.** The foundation is the test battery
  (survival, random-side). Regime merely gives the model better context.

---

## Variations

- **`janela_zscore=48`** — more reactive regime, detects short trends; increases
  noise, so require a higher `z_min` (2.5).
- **Add `ct_range` features** — include channel range and position within the
  channel to capture squeezes and expansions.
- **Regression instead of classification** — use `target_retorno` (log-return
  over N bars) to predict magnitude, not just direction.
- **Increase horizon** — `target_direcao` with `horizonte=10` or `15` to
  capture longer moves; requires more lags (7, 10) to align context.
