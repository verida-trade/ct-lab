# Recipe 07 — Custom LSTM Model (Python)

> **Level:** Advanced · **Premium** · **Prerequisites:** [Custom model](../../docs/06-ml/08-modelo-custom.en.md)

Train an LSTM with PyTorch via `modelo_custom` in the ML pipeline. The `uv` tool downloads dependencies (torch, numpy) automatically on first execution.

---

## Why LSTM?

| Model | When to use |
|---|---|
| GBDT (built-in) | Fast baseline, tabular features, no dependencies |
| MLP (built-in) | Non-linear, tabular features, with PyTorch |
| **LSTM (custom)** | Sequential data, temporal dependency between bars |

LSTM processes the feature window as a temporal sequence — each time step "remembers" previous ones via hidden state. In theory, it captures patterns that GBDT (which sees each bar independently) cannot see.

> **Warning:** LSTM is slower to train and more prone to overfitting than GBDT. Use as a second step — only after GBDT has already shown edge.

---

## Step 1 — Fetch series and features

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_features",
    "receita": "#{\"rsi\": rsi(close, 14), \"atr\": atr(high, low, close, 14)}"
  }
}
```

## Step 2 — LSTM Python script

```python
# /// script
# dependencies = ["torch==2.12.0", "numpy>=2.0"]
# ///

import torch
import torch.nn as nn
import numpy as np

class LSTMModel(nn.Module):
    def __init__(self, input_dim, hidden=32, classes=3):
        super().__init__()
        self.lstm = nn.LSTM(input_dim, hidden, batch_first=True)
        self.fc = nn.Linear(hidden, classes)

    def forward(self, x):
        _, (h, _) = self.lstm(x)
        return self.fc(h[-1])

def treinar(X, y, hp):
    # X: (n_samples, seq_len, input_dim) — temporal window
    # y: (n_samples,) — classes 0, 1, 2
    model = LSTMModel(X.shape[2], hp.get('hidden', 32))
    optimizer = torch.optim.Adam(model.parameters(), lr=hp.get('lr', 0.001))
    criterion = nn.CrossEntropyLoss()
    X_t = torch.tensor(X, dtype=torch.float32)
    y_t = torch.tensor(y, dtype=torch.long)
    for epoch in range(hp.get('epochs', 50)):
        optimizer.zero_grad()
        out = model(X_t)
        loss = criterion(out, y_t)
        loss.backward()
        optimizer.step()
    return model

def inferir(model, X):
    model.eval()
    with torch.no_grad():
        X_t = torch.tensor(X, dtype=torch.float32)
        return model(X_t).argmax(dim=1).numpy()
```

## Step 3 — Assemble ML pipeline

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_lstm",
    "nos": [
      {
        "id": "feats",
        "componente": { "op": "feature_set", "colunas": ["rsi", "atr"] },
        "entradas": ["ct://derived/btc_features"]
      },
      {
        "id": "lags",
        "componente": { "op": "gerar_lags", "colunas": ["rsi", "atr"], "lags": [1, 2, 3] },
        "entradas": ["$feats"]
      },
      {
        "id": "feats_lag",
        "componente": { "op": "aplicar_em_features" },
        "entradas": ["$lags", "$feats"]
      },
      {
        "id": "alvo",
        "componente": { "op": "target_direcao", "horizonte": 5, "limiar": 0.0 },
        "entradas": ["ct://series/binance/BTCUSDT/15m"]
      },
      {
        "id": "ds",
        "componente": { "op": "dataset" },
        "entradas": ["$feats_lag", "$alvo"]
      },
      {
        "id": "split",
        "componente": { "op": "split_simples", "treino_frac": 0.7 },
        "entradas": ["$ds"]
      },
      {
        "id": "scaler",
        "componente": { "op": "scaler_zscore" },
        "entradas": ["$split"]
      },
      {
        "id": "model",
        "componente": {
          "op": "modelo_custom",
          "script": "... Python script above ...",
          "deps": ["torch==2.12.0", "numpy>=2.0"],
          "hiperparametros": { "hidden": 32, "lr": 0.001, "epochs": 50 }
        },
        "entradas": ["$scaler"]
      },
      {
        "id": "evaluate",
        "componente": { "op": "avaliador" },
        "entradas": ["$model"]
      },
      {
        "id": "pred",
        "componente": { "op": "prever" },
        "entradas": ["$model", "$feats_lag"]
      }
    ],
    "modelo": "$model",
    "predicao": "$pred",
    "avaliacao": "$evaluate"
  }
}
```

## What each node does

| Node | `op` | Function |
|---|---|---|
| `feats` | `feature_set` | Packs RSI and ATR as matrix X |
| `lags` | `gerar_lags` | Creates 6 extra features: `rsi_lag1..3`, `atr_lag1..3` |
| `feats_lag` | `aplicar_em_features` | Reapplies lags to the feature set |
| `alvo` | `target_direcao` | Labels +1/0/-1 for 5 bars ahead |
| `ds` | `dataset` | Matches features + target by timestamp |
| `split` | `split_simples` | 70% train, 30% validation |
| `scaler` | `scaler_zscore` | Normalizes features to mean 0, std 1 |
| `model` | `modelo_custom` | Executes Python script, trains LSTM with PyTorch |
| `evaluate` | `avaliador` | Computes validation metrics |
| `pred` | `prever` | Materializes prediction at `ct://derived/btc_lstm_pred` |

> **Note:** `uv` downloads `torch` (~800MB) on first run. Deps travel in the `ModeloFitado` so `aplicar_modelo` can rebuild the environment at inference time.

## Variations

- **Larger hidden:** `hidden: 64` or `128` — more capacity, more overfit risk
- **More epochs:** `epochs: 100` with manual early stopping
- **Bidirectional:** add `nn.LSTM(..., bidirectional=True)` to the class
- **More features:** add Bollinger, Stochastic, or calendar features
