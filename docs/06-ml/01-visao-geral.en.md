# ML Pipeline Overview

> **Premium.** The ML DAG: feature_set → dataset → split → preprocessing → model → evaluate → predict → serving.

The `ct-mcp-server` ML pipeline is a **declarative DAG** of components. Each DAG node is a component with inputs referencing other nodes via `$<id>` or series URIs.

```
[feature_set] → [dataset] → [split] → [preprocessing] → [model] → [evaluate]
                                   ↓
                              [prediction] → ct://derived/<name>_pred
                                            ct://models/<name>
```

---

## `montar_esteira_ml`

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "my_model",
    "nos": [
      { "id": "features", "componente": {...}, "entradas": ["ct://derived/my_indicators"] },
      { "id": "target", "componente": {...}, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": {...}, "entradas": ["$features", "$target"] },
      { "id": "split", "componente": {...}, "entradas": ["$dataset"] },
      { "id": "scaler", "componente": {...}, "entradas": ["$split"] },
      { "id": "model", "componente": {...}, "entradas": ["$scaler"] },
      { "id": "evaluate", "componente": {...}, "entradas": ["$model"] }
    ],
    "modelo": "$model",
    "predicao": "$model",
    "avaliacao": "$evaluate"
  }
}
```

### Main parameters

| Field | Description |
|---|---|
| `name` | Model name → `ct://models/<name>` |
| `nos` | DAG nodes list (topological order) |
| `modelo` | `$<id>` ref to node whose Ajuste is the deliverable model |
| `predicao` | Optional ref to node whose Predição becomes `ct://derived/<name>_pred` |
| `avaliacao` | Optional ref to Evaluator node |
| `avalar_backtest` | Optional economic evaluation (bridge to `ct_backtest`) |

---

## Return

```json
{
  "model_uri": "ct://models/my_model",
  "pred_uri": "ct://derived/my_model_pred",
  "metricas": { "acuracia": 0.65, "f1": 0.58 },
  "validacao": { ... }
}
```

The model is persisted at `ct://models/<name>`. The prediction is materialized as a derived series at `ct://derived/<name>_pred` — can be used as an indicator in backtests or other pipelines.

---

## Component catalog

Always consult the live catalog:

**Resource:** `ct://ml/catalog`

---

> Next: [Feature engineering](./02-feature-engineering.en.md)
