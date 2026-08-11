# Built-in Models

> 6 model families ready to use — no Python needed.

| Family | Backend | Deps | Best for |
|---|---|---|---|
| `centroide` | stdlib (no deps) | none | Quick baseline, simple classification |
| `gbdt` | scikit-learn | `scikit-learn==1.9.0` | Tabular, boosting |
| `linear` | scikit-learn | `scikit-learn==1.9.0` | Simple regression/logistic |
| `random_forest` | scikit-learn | `scikit-learn==1.9.0` | Robustness, less overfit |
| `knn` | scikit-learn | `scikit-learn==1.9.0` | Non-parametric |
| `mlp` | torch | `torch==2.12.0` | Neural networks, non-linear |

---

## Usage

```json
{
  "componente": {
    "familia": "gbdt",
    "tarefa": "classificacao",
    "hiperparametros": { "n_estimators": 200, "max_depth": 5, "learning_rate": 0.1 }
  }
}
```

| Parameter | Default | Description |
|---|---|---|
| `familia` | required | Model family |
| `tarefa` | `classificacao` | `classificacao` or `regressao` |
| `hiperparametros` | — | Backend-specific hyperparameters |
| `deps` | — | Extra deps for `uv` environment |

---

> Next: [Custom Python model](./08-modelo-custom.en.md)
