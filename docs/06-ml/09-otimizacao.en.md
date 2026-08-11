# Hyperparameter Optimization

> Grid or random search to find the best hyperparameters.

```json
{
  "componente": {
    "estrategia": "grid",
    "familia": "gbdt",
    "tarefa": "classificacao",
    "grade": {
      "n_estimators": [50, 100, 200],
      "max_depth": [3, 4, 5],
      "learning_rate": [0.01, 0.05, 0.1]
    },
    "max_combos": 27
  }
}
```

| Parameter | Default | Description |
|---|---|---|
| `estrategia` | `grid` | `grid` (all) or `random` (sample) |
| `familia` | required | Model family |
| `grade` | required | Hyperparameter → list of values |
| `hiperparametros_base` | — | Fixed params combined with each grid point |
| `max_combos` | — | Max combinations (required in practice for `random`) |
| `seed` | 0 | Seed for `random` |
| `tarefa` | `classificacao` | Defines ranking metric (accuracy↑/R²↑) |

---

> Next: [Evaluation](./10-avaliacao.en.md)
