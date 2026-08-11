# Custom Transform (Python)

> `transformar_custom` — fit-dependent preprocessing via Python script.

Unlike `transformar_coluna` (stateless), `transformar_custom` learns parameters in training and reapplies them at serving.

## Contract

```python
def ajustar(X, hp):
    # Learns parameters from training (e.g., custom median for imputation)
    # Returns: (fit_model, X_transformed)
    ...

def transformar(fit_model, X):
    # Reapplies the learned fit to new data
    # Returns: X_transformed
    ...
```

## Usage

```json
{
  "componente": {
    "script": "import numpy as np\n\ndef ajustar(X, hp):\n    mediana = np.median(X, axis=0)\n    X_t = X - mediana\n    return {'mediana': mediana.tolist()}, X_t\n\ndef transformar(ajuste, X):\n    return X - np.array(ajuste['mediana'])",
    "deps": ["numpy>=2.0"]
  }
}
```

---

> Back to: [README](./README.en.md)
