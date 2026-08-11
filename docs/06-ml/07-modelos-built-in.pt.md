# Modelos Built-in

> 6 famílias de modelos prontas para uso — sem escrever Python.

| Família | Backend | Deps | Melhor para |
|---|---|---|---|
| `centroide` | stdlib (sem deps) | nenhuma | Baseline rápido, classificação simples |
| `gbdt` | scikit-learn | `scikit-learn==1.9.0` | Tabular, boosting |
| `linear` | scikit-learn | `scikit-learn==1.9.0` | Regressão/logística simples |
| `random_forest` | scikit-learn | `scikit-learn==1.9.0` | Robustez, menos overfit |
| `knn` | scikit-learn | `scikit-learn==1.9.0` | Não-paramétrico |
| `mlp` | torch | `torch==2.12.0` | Redes neurais, não-linear |

---

## Uso

```json
{
  "componente": {
    "familia": "gbdt",
    "tarefa": "classificacao",
    "hiperparametros": {
      "n_estimators": 200,
      "max_depth": 5,
      "learning_rate": 0.1
    }
  }
}
```

| Parâmetro | Default | Descrição |
|---|---|---|
| `familia` | obrigatório | `centroide`/`gbdt`/`linear`/`random_forest`/`knn`/`mlp` |
| `tarefa` | `classificacao` | `classificacao` ou `regressao` |
| `hiperparametros` | — | Hiperparâmetros do backend (formato dele) |
| `deps` | — | Deps extras para o ambiente `uv` |

---

## Exemplo: GBDT para direção

```json
{
  "id": "modelo",
  "componente": {
    "familia": "gbdt",
    "tarefa": "classificacao",
    "hiperparametros": {
      "n_estimators": 100,
      "max_depth": 4,
      "learning_rate": 0.05
    }
  },
  "entradas": ["$scaler"]
}
```

---

## Deps em camadas

As dependências são resolvidas em camadas:

1. **Base**: definida por `familia` (`gbdt` → scikit-learn, `mlp` → torch)
2. **Extras**: `deps: ["scipy==1.14.0"]` soma à base
3. **Override**: `deps: ["scikit-learn==1.8.0"]` sobrescreve a versão base

O `uv` resolve as deps para a versão pinada do Python (`CT_MCP_ML_PYTHON`, default `3.14.5`).

---

> Próximo: [Modelo custom Python](./08-modelo-custom.pt.md)
