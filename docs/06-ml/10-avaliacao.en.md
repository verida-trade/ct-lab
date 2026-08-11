# Evaluation

> Measure model quality with classification, regression, clustering or anomaly metrics.

## `avaliar`

Evaluates prediction vs actual target:

```json
{ "componente": {}, "entradas": ["$model"] }
```

Returns: accuracy, F1, precision, recall, confusion matrix (classification); R², MAE, MSE (regression).

## `avaliar_cv`

Cross-validated evaluation with multiple folds.

## `avaliar_regressao`

For regression models.

## `avaliar_custom` (Python)

Custom metric:

```json
{
  "componente": {
    "script": "def avaliar(y_pred, y_true, hp):\n    correct = sum(1 for p, t in zip(y_pred, y_true) if p == t)\n    return {'custom_acc': correct / len(y_true)}"
  }
}
```

---

> Next: [Serving](./11-serving.en.md)
