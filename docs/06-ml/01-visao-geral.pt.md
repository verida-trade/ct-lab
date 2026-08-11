# Visão Geral da Esteira ML

> **Premium.** O DAG de ML: feature_set → dataset → split → preprocessing → modelo → avaliar → prever → serving.

A esteira ML do `ct-mcp-server` é um **DAG declarativo** de componentes. Cada nó do DAG é um componente (feature_set, target, split, scaler, modelo, etc.) com entradas que referenciam outros nós via `$<id>` ou URIs de séries.

```
[feature_set] → [dataset] → [split] → [preprocessing] → [modelo] → [avaliar]
                                   ↓
                              [predicao] → ct://derived/<name>_pred
                                          ct://models/<name>
```

---

## `montar_esteira_ml`

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "meu_modelo",
    "nos": [
      { "id": "features", "componente": {...}, "entradas": ["ct://derived/meus_indicadores"] },
      { "id": "target", "componente": {...}, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": {...}, "entradas": ["$features", "$target"] },
      { "id": "split", "componente": {...}, "entradas": ["$dataset"] },
      { "id": "scaler", "componente": {...}, "entradas": ["$split"] },
      { "id": "modelo", "componente": {...}, "entradas": ["$scaler"] },
      { "id": "avaliar", "componente": {...}, "entradas": ["$modelo"] }
    ],
    "modelo": "$modelo",
    "predicao": "$modelo",
    "avaliacao": "$avaliar"
  }
}
```

### Parâmetros principais

| Campo | Descrição |
|---|---|
| `name` | Nome do modelo → `ct://models/<name>` |
| `nos` | Lista de nós do DAG (ordem topológica) |
| `modelo` | Referência `$<id>` ao nó cujo Ajuste é o modelo entregável |
| `predicao` | Referência opcional ao nó cuja Predição vira `ct://derived/<name>_pred` |
| `avaliacao` | Referência opcional ao nó Avaliador |
| `avalar_backtest` | Avaliação econômica opcional (ponte para `ct_backtest`) |

---

## Retorno

```json
{
  "model_uri": "ct://models/meu_modelo",
  "pred_uri": "ct://derived/meu_modelo_pred",
  "metricas": { "acuracia": 0.65, "f1": 0.58, ... },
  "validacao": { ... }
}
```

O modelo é persistido em `ct://models/<name>`. A predição é materializada como série derived em `ct://derived/<name>_pred` — pode ser usada como indicador em backtests ou em outras esteiras.

---

## Catálogo de componentes

Consulte sempre o catálogo vivo:

**Resource:** `ct://ml/catalog`

Componentes disponíveis: feature_set, gerar_lags, features_calendario, transformar_coluna, interagir_colunas, target_direcao, target_retorno, target_custom, target_conjunto, split_holdout, split_walk_forward, split_purged_kfold, split_custom, dataset, imputar, preencher_temporal, winsorize, scaler_zscore, minmax, robust, maxabs, selecionar_correlacao, selecionar_variancia, reduzir_pca, modelo (centroide/gbdt/linear/random_forest/knn/mlp), modelo_custom, transformar_custom, avaliar, avaliar_cv, avaliar_regressao, avaliar_custom, gerar_custom, otimizar_hiperparametros, analisar_dataset, empilhar_datasets.

---

> Próximo: [Feature engineering](./02-feature-engineering.pt.md)
