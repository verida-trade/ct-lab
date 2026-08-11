# Otimização de Hiperparâmetros

> Busca em grade (grid) ou aleatória (random) para encontrar os melhores hiperparâmetros.

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
    "hiperparametros_base": {},
    "max_combos": 27
  }
}
```

| Parâmetro | Default | Descrição |
|---|---|---|
| `estrategia` | `grid` | `grid` (todas) ou `random` (amostra) |
| `familia` | obrigatório | Família do modelo |
| `grade` | obrigatório | Hiperparâmetro → lista de valores |
| `hiperparametros_base` | — | Hiperparâmetros fixos combinados com cada ponto |
| `max_combos` | — | Teto de combinações (obrigatório na prática para `random`) |
| `seed` | 0 | Semente para `random` |
| `tarefa` | `classificacao` | Define a métrica de ranking (acurácia↑/R²↑) |

---

> Próximo: [Avaliação](./10-avaliacao.pt.md)
