# Recipe 07 — Custom LSTM Model (Python)

> **Level:** Advanced · **Premium** · **Prerequisites:** [Custom model](../docs/06-ml/08-modelo-custom.en.md)

Train an LSTM with PyTorch via `modelo_custom`.

## Python script

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

## Pipeline

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_lstm",
    "nos": [
      { "id": "features", "componente": { "colunas": ["rsi", "atr"] }, "entradas": ["ct://derived/btc_features"] },
      { "id": "target", "componente": { "coluna": "close", "horizonte": 5 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": {}, "entradas": ["$features", "$target"] },
      { "id": "split", "componente": { "treino_frac": 0.7 }, "entradas": ["$dataset"] },
      { "id": "scaler", "componente": {}, "entradas": ["$split"] },
      {
        "id": "model",
        "componente": {
          "script": "... script above ...",
          "deps": ["torch==2.12.0"],
          "hiperparametros": { "hidden": 32, "lr": 0.001, "epochs": 50 }
        },
        "entradas": ["$scaler"]
      },
      { "id": "evaluate", "componente": {}, "entradas": ["$model"] }
    ],
    "modelo": "$model",
    "predicao": "$model",
    "avaliacao": "$evaluate"
  }
}
```

> `uv` downloads torch on first run. Deps travel in `ModeloFitado` for `aplicar_modelo` to rebuild the environment.
