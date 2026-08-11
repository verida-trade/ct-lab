# Machine Learning

> Esteiras de ML: feature engineering, datasets, splits, modelos, avaliação e serving.

## Documentos

| # | Documento | O que cobre |
|---|---|---|
| 1 | [Visão geral](./01-visao-geral.pt.md) | DAG da esteira ML: features→dataset→split→modelo→avaliar→prever |
| 2 | [Feature engineering](./02-feature-engineering.pt.md) | `gerar_lags`, `features_calendario`, `transformar_coluna`, `interagir_colunas` |
| 3 | [Feature sets](./03-feature-sets.pt.md) | Puxar colunas de qualquer série como features |
| 4 | [Targets](./04-targets.pt.md) | `target_direcao`, `target_retorno`, `target_custom`, `target_conjunto` |
| 5 | [Splits](./05-splits.pt.md) | `split_holdout`, `split_walk_forward`, `split_purged_kfold`, `split_custom` |
| 6 | [Preprocessing](./06-preprocessing.pt.md) | Imputar, scalers, seleção, PCA |
| 7 | [Modelos built-in](./07-modelos-built-in.pt.md) | `centroide`, `gbdt`, `linear`, `random_forest`, `knn`, `mlp` |
| 8 | [Modelo custom Python](./08-modelo-custom.pt.md) | `modelo_custom` via `uv` — treinar/inferir |
| 9 | [Otimização de hiperparâmetros](./09-otimizacao.pt.md) | `otimizar_hiperparametros` — grid/random |
| 10 | [Avaliação](./10-avaliacao.pt.md) | `avaliar`, `avaliar_cv`, métricas |
| 11 | [Serving](./11-serving.pt.md) | `aplicar_modelo` — reaplicar sem re-treinar |
| 12 | [Análise de dataset (EDA)](./12-eda.pt.md) | `analisar_dataset` |
| 13 | [Empilhamento de datasets](./13-empilhar.pt.md) | Treino multi-série/multi-TF |
| 14 | [Ponte backtest↔ML](./14-ponte-backtest.pt.md) | `avalar_backtest` — métricas `bt_*` |
| 15 | [Ambiente Python `uv`](./15-uv.pt.md) | Configuração, deps, version pinning |
| 16 | [Catálogo de componentes](./16-catalogo.pt.md) | `ct://ml/catalog` |
| 17 | [Modelo de regime](./17-regime.pt.md) | `ct_regime` + HMM + `aplicar_modelo` |
| 18 | [Transformação custom](./18-transformar-custom.pt.md) | `transformar_custom` — pré-proc fit-dependente |

> Versão em inglês: [README.en.md](./README.en.md)
