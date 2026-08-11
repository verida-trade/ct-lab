# Avaliação

> Meça a qualidade do modelo com métricas de classificação, regressão, clustering ou anomalia.

## `avaliar`

Avalia predição vs target real:

```json
{ "componente": {}, "entradas": ["$modelo"] }
```

Retorna: acurácia, F1, precision, recall, matriz de confusão (classificação); R², MAE, MSE (regressão).

## `avaliar_cv`

Avaliação cruzada com múltiplos folds:

```json
{ "componente": { "familia": "gbdt", "tarefa": "classificacao" } }
```

## `avaliar_regressao`

Para modelos de regressão:

```json
{ "componente": {} }
```

## `avaliar_custom` (Python)

Métrica custom:

```json
{
  "componente": {
    "script": "def avaliar(y_pred, y_true, hp):\n    correct = sum(1 for p, t in zip(y_pred, y_true) if p == t)\n    return {'custom_acc': correct / len(y_true)}"
  }
}
```

---

> Próximo: [Serving](./11-serving.pt.md)
