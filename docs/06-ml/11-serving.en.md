# Serving — `aplicar_modelo`

> Reapply a trained model without retraining. The model is an opaque artifact at `ct://models/<name>`.

## Call

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "ct://models/my_model",
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "probas": true
  }
}
```

| Parameter | Description |
|---|---|
| `modelo` | Trained model URI |
| `fonte` | Raw or derived URI (anchor = raw from chain) |
| `probas` | `true` → materialize probability PER CLASS (columns `p_<class>`) instead of argmax class |

## Return

The prediction is materialized as a derived series aligned to the anchor's timeframe:

```
ct://derived/<model_name>_pred
```

With `probas: true`, generates columns `p_0`, `p_1`, `p_2` (one per class).

## Reapplication

`aplicar_modelo` rebuilds the `uv` environment with the same deps, loads the model and reapplies the same adjustments (scalers, transforms) as in training. **Does not retrain**.

---

> Next: [EDA](./12-eda.en.md)
