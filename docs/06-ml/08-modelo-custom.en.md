# Custom Model (Python)

> Bring your own model — LSTM, Transformer, XGBoost, anything. Via `uv` with pinned deps.

Use `modelo_custom` to train and infer with any Python architecture.

---

## Structure

```json
{
  "componente": {
    "script": "import numpy as np\nfrom sklearn.ensemble import GradientBoostingClassifier\n\ndef treinar(X, y, hp):\n    ...\n\ndef inferir(model, X):\n    ...",
    "deps": ["scikit-learn==1.9.0"],
    "hiperparametros": { "n_estimators": 100 }
  }
}
```

### Via URI

```json
{ "componente": { "uri": "file:///path/to/my_model.py", "deps": ["torch==2.12.0"] } }
```

---

## Python contract

The script must define two functions:

```python
def treinar(X, y, hp):
    # X: numpy array (n_samples, n_features)
    # y: numpy array (n_samples,)
    # hp: dict of hyperparameters
    # Returns: model object (anything pickleable)
    ...

def inferir(model, X):
    # model: object returned by treinar
    # X: numpy array (n_samples, n_features)
    # Returns: numpy array of predictions (n_samples,)
    ...
```

---

> Next: [Hyperparameter optimization](./09-otimizacao.en.md)
