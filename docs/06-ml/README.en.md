# Machine Learning

> ML pipelines: feature engineering, datasets, splits, models, evaluation and serving.

## Documents

| # | Document | Covers |
|---|---|---|
| 1 | [Overview](./01-visao-geral.en.md) | ML pipeline DAG: features→dataset→split→model→evaluate→predict |
| 2 | [Feature engineering](./02-feature-engineering.en.md) | `gerar_lags`, `features_calendario`, `transformar_coluna`, `interagir_colunas` |
| 3 | [Feature sets](./03-feature-sets.en.md) | Pull columns from any series as features |
| 4 | [Targets](./04-targets.en.md) | `target_direcao`, `target_retorno`, `target_custom`, `target_conjunto` |
| 5 | [Splits](./05-splits.en.md) | `split_holdout`, `split_walk_forward`, `split_purged_kfold`, `split_custom` |
| 6 | [Preprocessing](./06-preprocessing.en.md) | Impute, scalers, selection, PCA |
| 7 | [Built-in models](./07-modelos-built-in.en.md) | `centroide`, `gbdt`, `linear`, `random_forest`, `knn`, `mlp` |
| 8 | [Custom Python model](./08-modelo-custom.en.md) | `modelo_custom` via `uv` — train/infer |
| 9 | [Hyperparameter optimization](./09-otimizacao.en.md) | `otimizar_hiperparametros` — grid/random |
| 10 | [Evaluation](./10-avaliacao.en.md) | `avaliar`, `avaliar_cv`, metrics |
| 11 | [Serving](./11-serving.en.md) | `aplicar_modelo` — reapply without retraining |
| 12 | [Dataset analysis (EDA)](./12-eda.en.md) | `analisar_dataset` |
| 13 | [Dataset stacking](./13-empilhar.en.md) | Multi-series/multi-TF training |
| 14 | [Backtest↔ML bridge](./14-ponte-backtest.en.md) | `avalar_backtest` — `bt_*` metrics |
| 15 | [Python environment `uv`](./15-uv.en.md) | Config, deps, version pinning |
| 16 | [Component catalog](./16-catalogo.en.md) | `ct://ml/catalog` |
| 17 | [Regime model](./17-regime.en.md) | `ct_regime` + HMM + `aplicar_modelo` |
| 18 | [Custom transform](./18-transformar-custom.en.md) | `transformar_custom` — fit-dependent pre-proc |

> Portuguese version: [README.pt.md](./README.pt.md)
