# Receita 07 — Modelo Custom LSTM (Python)

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Modelo custom](../../docs/06-ml/08-modelo-custom.pt.md)

Treine uma LSTM com PyTorch via `modelo_custom` na esteira ML. O `uv` baixa as dependências (torch, numpy) automaticamente na primeira execução.

---

## Por que LSTM?

| Modelo | Quando usar |
|---|---|
| GBDT (built-in) | Baseline rápido, features tabulares, sem dependências |
| MLP (built-in) | Não-linear, features tabulares, com PyTorch |
| **LSTM (custom)** | Dados sequenciais, dependência temporal entre barras |

A LSTM processa a janela de features como uma sequência temporal — cada passo de tempo "lembra" dos anteriores via estado oculto. Em teoria, captura padrões que GBDT (que vê cada barra isoladamente) não enxerga.

> **Atenção:** LSTM é mais lento para treinar e mais propenso a overfitting que GBDT. Use como segundo passo — só depois que GBDT já mostrou edge.

---

## Passo 1 — Buscar série e features

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

## Passo 2 — Script Python da LSTM

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
    # X: (n_samples, seq_len, input_dim) — janela temporal
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

## Passo 3 — Montar esteira ML

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
          "script": "... script Python acima ...",
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

## O que cada nó faz

| Nó | `op` | Função |
|---|---|---|
| `feats` | `feature_set` | Empacota RSI e ATR como matriz X |
| `lags` | `gerar_lags` | Cria 6 features extras: `rsi_lag1..3`, `atr_lag1..3` |
| `feats_lag` | `aplicar_em_features` | Reaplica lags ao feature_set |
| `alvo` | `target_direcao` | Rótula +1/0/-1 para 5 barras à frente |
| `ds` | `dataset` | Casa features + target por timestamp |
| `split` | `split_simples` | 70% treino, 30% validação |
| `scaler` | `scaler_zscore` | Normaliza features para média 0, desvio 1 |
| `model` | `modelo_custom` | Executa script Python, treina LSTM com PyTorch |
| `evaluate` | `avaliador` | Calcula métricas de validação |
| `pred` | `prever` | Materializa predição em `ct://derived/btc_lstm_pred` |

> **Nota:** O `uv` baixa `torch` (~800MB) na primeira execução. As deps viajam no `ModeloFitado` para o `aplicar_modelo` reconstruir o ambiente na inferência.

## Variações

- **Hidden maior:** `hidden: 64` ou `128` — mais capacidade, mais risco de overfit
- **Mais epochs:** `epochs: 100` com early stopping manual
- **Bidirectional:** adicionar `nn.LSTM(..., bidirectional=True)` na classe
- ** Mehr features:** adicionar Bollinger, Stochastic, ou features de calendário
