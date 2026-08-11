# Modelo Custom (Python)

> Traga seu próprio modelo — LSTM, Transformer, XGBoost, o que for. Via `uv` com deps pinadas.

Use `modelo_custom` para treinar e inferir com qualquer arquitetura Python.

---

## Estrutura

```json
{
  "componente": {
    "script": "import numpy as np\nfrom sklearn.ensemble import GradientBoostingClassifier\n\ndef treinar(X, y, hp):\n    model = GradientBoostingClassifier(\n        n_estimators=hp.get('n_estimators', 100),\n        max_depth=hp.get('max_depth', 3)\n    )\n    model.fit(X, y)\n    return model\n\ndef inferir(model, X):\n    return model.predict(X)",
    "deps": ["scikit-learn==1.9.0"],
    "hiperparametros": { "n_estimators": 100, "max_depth": 3 }
  }
}
```

### Via URI

```json
{
  "componente": {
    "uri": "file:///path/to/meu_modelo.py",
    "deps": ["torch==2.12.0"]
  }
}
```

---

## Contrato Python

O script deve definir duas funções:

```python
def treinar(X, y, hp):
    # X: numpy array (n_samples, n_features)
    # y: numpy array (n_samples,)
    # hp: dict de hiperparâmetros
    # Retorna: o objeto modelo (qualquer coisa pickleable)
    ...

def inferir(model, X):
    # model: o objeto retornado por treinar
    # X: numpy array (n_samples, n_features)
    # Retorna: numpy array de predições (n_samples,)
    ...
```

---

## Exemplo: LSTM com PyTorch

```python
# /// script
# dependencies = ["torch==2.12.0", "numpy>=2.0"]
# ///

import torch
import torch.nn as nn
import numpy as np

class LSTMModel(nn.Module):
    def __init__(self, input_dim, hidden=64):
        super().__init__()
        self.lstm = nn.LSTM(input_dim, hidden, batch_first=True)
        self.fc = nn.Linear(hidden, 3)
    
    def forward(self, x):
        _, (h, _) = self.lstm(x)
        return self.fc(h[-1])

def treinar(X, y, hp):
    model = LSTMModel(X.shape[2], hp.get('hidden', 64))
    # ... training loop ...
    return model

def inferir(model, X):
    model.eval()
    with torch.no_grad():
        return model(torch.tensor(X, dtype=torch.float32)).argmax(dim=1).numpy()
```

> O `uv` baixa as deps na primeira execução. As deps viajam no `ModeloFitado` para o `prever` montar o mesmo ambiente.

---

> Próximo: [Otimização de hiperparâmetros](./09-otimizacao.pt.md)
