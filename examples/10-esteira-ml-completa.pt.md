# 10 — Esteira ML Completa: Do Feature ao Backtest

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Regime + Modelo](./09-regime-modelo.pt.md), [Modelo GBDT](./06-modelo-gbdt.pt.md), [Modelo MLP](./07-modelo-lstm.pt.md)

Você já treinou GBDT (exemplo 06), MLP (exemplo 07), detectou regime (exemplo 09) e mediu microestrutura (exemplo 08). Cada exemplo usou a esteira de ML, mas sempre numa versão simplificada — `feature_set → dataset → split → modelo → prever`. Agora é hora de montar a esteira **completa**: com lags, winsorização, imputação, scaling, e avaliação econômicavia backtest — tudo num único DAG declarativo.

A pergunta central: **preprocessamento melhora o modelo, ou só complica?**

> **Esteira de ML** é um DAG (grafo acíclico direcionado) de componentes. Cada componente recebe artefatos (FeatureSet, Dataset, Ajuste, Predicao) e produz outro. A esteira resolve a topologia em runtime — você declata o grafo, ela executa em ordem topológica.

---

## O problema

No exemplo 06, o GBDT simples teve accuracy de 70.6% e superou taxas: +$12.447 de PnL líquido. Mas aquela esteira não tinha nenhum preprocessamento — features cruas diretamente no modelo. A questão natural:

1. **Winsorização** (clipar outliers nos quantis 1%/99%) remove ruído extremo → o modelo aprende padrões mais limpos?
2. **Imputação** (preencher NaN com a média) recupera linhas perdidas no warmup dos lags → mais dados de treino?
3. **Z-Score scaling** (normalizar para média 0, desvio 1) equaliza a escala das features → o GBDT converge melhor?
4. **Lags de 5 features** (RSI, MACD, ADX, BOP, volume em t-1, t-2, t-3) dão memória temporal ao modelo → melhora previsão?

A esteira completa é o teste empírico dessas hipóteses.

---

## Passo 1 — Construir features avançadas

### 1.1 — Pipeline com 14 indicadores + BOP

O pipeline de features do exemplo 06 tinha 8 indicadores. Aqui vamos usar 14, adicionando BOP (microestrutura), SMA20/SMA50 (tendência), e Bollinger bands (volatilidade):

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "r10_full_feats",
    "output": "$features",
    "steps": [
      { "op": "rsi", "id": "rsi", "source": "$anchor", "column": "close", "period": 14 },
      { "op": "macd", "id": "macd", "source": "$anchor", "column": "close", "fast": 12, "slow": 26, "signal": 9 },
      { "op": "adx", "id": "adx", "source": "$anchor", "period": 14 },
      { "op": "bollinger", "id": "boll", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "atr", "id": "atr", "source": "$anchor", "period": 14 },
      { "op": "bop", "id": "bop", "source": "$anchor" },
      { "op": "sma", "id": "sma20", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "sma", "id": "sma50", "source": "$anchor", "column": "close", "period": 50 },
      {
        "op": "compose",
        "id": "features",
        "columns": [
          { "as_column": "rsi", "source": "$rsi", "source_column": "rsi" },
          { "as_column": "macd", "source": "$macd", "source_column": "macd" },
          { "as_column": "macd_signal", "source": "$macd", "source_column": "signal" },
          { "as_column": "adx", "source": "$adx", "source_column": "adx" },
          { "as_column": "plus_di", "source": "$adx", "source_column": "plus_di" },
          { "as_column": "minus_di", "source": "$adx", "source_column": "minus_di" },
          { "as_column": "bb_upper", "source": "$boll", "source_column": "upper" },
          { "as_column": "bb_lower", "source": "$boll", "source_column": "lower" },
          { "as_column": "bb_middle", "source": "$boll", "source_column": "middle" },
          { "as_column": "atr", "source": "$atr", "source_column": "atr" },
          { "as_column": "bop", "source": "$bop", "source_column": "bop" },
          { "as_column": "sma20", "source": "$sma20", "source_column": "sma" },
          { "as_column": "sma50", "source": "$sma50", "source_column": "sma" },
          { "as_column": "volume", "source": "$anchor", "source_column": "volume" },
          { "as_column": "close", "source": "$anchor", "source_column": "close" }
        ]
      }
    ]
  }
}
```

> Série resultante: `ct://derived/r10_full_feats` com 1712 linhas e 15 colunas (14 indicadores + close).

---

## Passo 2 — Esteira simples (baseline)

Antes da esteira completa, vamos建立 a baseline — a esteira mínima, igual à do exemplo 06, mas com mais features e lags de 4 features:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "r10_simples",
    "nos": [
      {
        "id": "feats",
        "componente": { "op": "feature_set",
          "colunas": ["rsi","macd","macd_signal","adx","plus_di","minus_di","bb_upper","bb_lower","bb_middle","atr","bop","sma20","sma50","volume"] },
        "entradas": ["ct://derived/r10_full_feats"]
      },
      {
        "id": "lags",
        "componente": { "op": "gerar_lags", "colunas": ["rsi","macd","adx","bop"], "lags": [1,2,3] },
        "entradas": ["$feats"]
      },
      {
        "id": "applied",
        "componente": { "op": "aplicar_em_features" },
        "entradas": ["$lags", "$feats"]
      },
      {
        "id": "alvo",
        "componente": { "op": "target_direcao", "horizonte": 1 },
        "entradas": ["ct://series/binance/BTCUSDT/15m"]
      },
      {
        "id": "ds",
        "componente": { "op": "dataset" },
        "entradas": ["$applied", "$alvo"]
      },
      {
        "id": "split",
        "componente": { "op": "split_walk_forward", "n_folds": 4, "treino_inicial_frac": 0.5 },
        "entradas": ["$ds"]
      },
      {
        "id": "modelo",
        "componente": { "op": "modelo", "familia": "gbdt",
          "hiperparametros": { "max_depth": 3, "n_estimators": 100, "learning_rate": 0.1 } },
        "entradas": ["$ds", "$split"]
      },
      {
        "id": "pred",
        "componente": { "op": "prever" },
        "entradas": ["$modelo", "$applied"]
      },
      {
        "id": "aval",
        "componente": { "op": "avaliar" },
        "entradas": ["$pred", "$ds", "$split"]
      }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$aval"
  }
}
```

### Métricas de validação (walk-forward, 4 folds)

| Métrica | Valor |
|---|---|
| Acurácia | 50.2% |
| F1 macro | 0.502 |
| Precisão macro | 0.502 |
| Revocação macro | 0.502 |
| N validação | 215 |

> **Acurácia de 50.2% é marginalmente melhor que aleatório** (33.3% para 3 classes). O modelo simples não está prevendo direção — está prevendo a classe majoritária.

### Backtest

| Métrica | Com fee 0.1% | Sem fee |
|---|---|---|
| **PnL** | −$28.164 | +$74.059 |
| **PnL bruto** | +$74.089 | +$74.089 |
| **Fees** | $102.095 | 0 |
| **Trades** | 795 | 795 |
| **Win rate** | 24.8% | 89.4% |
| **Profit factor** | 0.50 | 14.17 |
| **Sharpe** | 0.005 | 0.466 |
| **Exposição** | 99.7% | 99.7% |

> **O que aconteceu**: sem taxas, a estratégia tem win rate de 89.4% e profit factor de 14.17 — parece extraordinário. Mas são 795 trades em 1712 candles (46% de turnover): o modelo prevê direção quase sempre e o backtest executa quase todos os candles. Com taxas, $102k em fees destroem tudo. A acurácia de 50.2% confirma: o modelo não tem edge real, apenas overfits ao conjunto de treino.

---

## Passo 3 — Esteira completa (com preprocessamento)

Agora a esteira completa: tudo que a simples tem, mais **winsorização, imputação, z-score scaling**, e um **limiar** no target para filtrar movimentos mínimos:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "r10_full",
    "nos": [
      {
        "id": "feats",
        "componente": { "op": "feature_set",
          "colunas": ["rsi","macd","macd_signal","adx","plus_di","minus_di","bb_upper","bb_lower","bb_middle","atr","bop","sma20","sma50","volume"] },
        "entradas": ["ct://derived/r10_full_feats"]
      },
      {
        "id": "lags",
        "componente": { "op": "gerar_lags", "colunas": ["rsi","macd","adx","bop","volume"], "lags": [1,2,3] },
        "entradas": ["$feats"]
      },
      {
        "id": "applied",
        "componente": { "op": "aplicar_em_features" },
        "entradas": ["$lags", "$feats"]
      },
      {
        "id": "alvo",
        "componente": { "op": "target_direcao", "horizonte": 1, "limiar": 0.001 },
        "entradas": ["ct://series/binance/BTCUSDT/15m"]
      },
      {
        "id": "ds",
        "componente": { "op": "dataset", "manter_features_nan": true },
        "entradas": ["$applied", "$alvo"]
      },
      {
        "id": "split",
        "componente": { "op": "split_walk_forward", "n_folds": 4, "treino_inicial_frac": 0.5 },
        "entradas": ["$ds"]
      },
      {
        "id": "winso",
        "componente": { "op": "winsorize", "q_inf": 0.01, "q_sup": 0.99 },
        "entradas": ["$ds", "$split"]
      },
      {
        "id": "ds_w",
        "componente": { "op": "aplicar_winsorize" },
        "entradas": ["$winso", "$ds"]
      },
      {
        "id": "imp",
        "componente": { "op": "imputar", "estrategia": "media" },
        "entradas": ["$ds_w", "$split"]
      },
      {
        "id": "ds_i",
        "componente": { "op": "aplicar_imputacao" },
        "entradas": ["$imp", "$ds_w"]
      },
      {
        "id": "scaler",
        "componente": { "op": "scaler_zscore" },
        "entradas": ["$ds_i", "$split"]
      },
      {
        "id": "ds_s",
        "componente": { "op": "aplicar_scaler" },
        "entradas": ["$scaler", "$ds_i"]
      },
      {
        "id": "modelo",
        "componente": { "op": "modelo", "familia": "gbdt",
          "hiperparametros": { "max_depth": 3, "n_estimators": 100, "learning_rate": 0.1 } },
        "entradas": ["$ds_s", "$split"]
      },
      {
        "id": "pred",
        "componente": { "op": "prever" },
        "entradas": ["$modelo", "$applied"]
      },
      {
        "id": "aval",
        "componente": { "op": "avaliar" },
        "entradas": ["$pred", "$ds_s", "$split"]
      }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$aval"
  }
}
```

### Anatomia da esteira (15 nós)

```
feats ──→ lags ──→ applied ──→ ds ──→ split
  │                                        │
  │                                        ├─→ winso ──→ ds_w ──┐
  │                                        │                     │
  │                                        ├─→ imp ──→ ds_i ────┤
  │                                        │                     │
  │                                        └─→ scaler ──→ ds_s ──┤
  │                                                              │
  └──────────────────────────────────────────────────→ pred ←── modelo ←─┘
                                                         │
                                                         └──→ aval
```

| Nó | Componente | O que faz |
|---|---|---|
| `feats` | `feature_set` | Seleciona 14 colunas da série de features |
| `lags` | `gerar_lags` | Gera lags (1,2,3) de 5 features → +15 colunas |
| `applied` | `aplicar_em_features` | Reaplica lags ao FeatureSet |
| `alvo` | `target_direcao` | Direção do retorno (−1, 0, +1) com limiar 0.1% |
| `ds` | `dataset` | Junta features + target, mantém NaN |
| `split` | `split_walk_forward` | 4 folds walk-forward, 50% treino inicial |
| `winso` | `winsorize` | Aprende limites 1%/99% no treino |
| `ds_w` | `aplicar_winsorize` | Clipa outliers aos limites aprendidos |
| `imp` | `imputar` | Aprende média de cada coluna no treino |
| `ds_i` | `aplicar_imputacao` | Preenche NaN com a média do treino |
| `scaler` | `scaler_zscore` | Aprende média/desvio por coluna no treino |
| `ds_s` | `aplicar_scaler` | Normaliza features para z-score |
| `modelo` | `modelo` (gbdt) | Treina GBDT (max_depth=3, 100 árvores) |
| `pred` | `prever` | Aplica modelo → predição por barra |
| `aval` | `avaliar` | Computa acurácia/F1 no fold de validação |

> **Anti-leakage**: `winsorize`, `imputar`, e `scaler` são `EstimadorFitDependente` — recebem `(dataset, split)` e fazem **fit só no treino**. Os `aplicar_*` aplicam o ajuste a todo o dataset usando os parâmetros do treino. Sem vazamento de informação do teste para o treino.

### Métricas de validação

| Métrica | Esteira simples | Esteira completa |
|---|---|---|
| Acurácia | 50.2% | 18.7% |
| F1 macro | 0.502 | 0.108 |
| Precisão macro | 0.502 | 0.228 |
| N validação | 215 | 214 |

> **A acurácia despenca de 50.2% para 18.7%!** Isso parece desastroso, mas é enganoso. A esteira simples prevê quase todas as barras (795 trades) — tem acurácia inflada pelo viés da classe majoritária. A esteira completa, com limiar de 0.1%, prevê quase tudo como classe 0 (neutro) — a acurácia por classe despenca, mas o modelo **só opera quando tem convicção**.

### Backtest

| Métrica | Simples com fee | Completa com fee | Simples sem fee | Completa sem fee |
|---|---|---|---|---|
| **PnL** | −$28.164 | −$408 | +$74.059 | +$746 |
| **PnL bruto** | +$74.089 | +$530 | +$74.089 | +$530 |
| **Fees** | $102.095 | $1.026 | 0 | 0 |
| **Trades** | 795 | 8 | 795 | 8 |
| **Win rate** | 24.8% | 25.0% | 89.4% | 50.0% |
| **Profit factor** | 0.50 | 0.84 | 14.17 | 1.21 |
| **Sharpe** | 0.005 | 0.002 | 0.466 | 0.009 |
| **Calmar** | −0.36 | −1.95 | ≈∞ | 13.44 |
| **Max DD** | 281.6% | 29.5% | 1.8% | 25.1% |
| **Exposição** | 99.7% | 99.3% | 99.7% | 99.3% |
| **Expectativa** | −$35/trade | −$62/trade | +$93/trade | +$66/trade |
| **Retorno anual.** | −100% | −57.4% | ≈∞ | +337% |

---

## Passo 4 — Análise: o preprocessamento funcionou?

### O que a esteira completa conseguiu

**Reduziu turnover de 795 para 8 trades** — uma redução de 99%. As fees caíram de $102k para $1k — um fator de 100x. O modelo deixou de prever quase todas as barras e passou a prever apenas quando tem convicção (classe ≠ 0).

**Profit factor com fee melhorou**: de 0.50 (simples) para 0.84 (completa). Ainda não supera taxas, mas está muito mais perto.

### O que não conseguiu

**Não superou taxas**. O PnL com fee é −$408. As 8 trades geraram $530 de PnL bruto contra $1.026 de fees. O edge existe (PF 1.21 sem fee), mas não é grande o suficiente para superar 0.1% de taxa por lado.

**A acurácia caiu** porque o limiar (0.1%) classificou muitos targets como classe 0 (neutro), e o modelo prevê a classe majoritária. Mas isso é **feature, não bug** — o modelo só prevê direção quando o retorno esperado é > 0.1%, que é exatamente quando vale a pena operar.

### O que faltaria

O GBDT do exemplo 06 superou taxas (+$12.447) porque usou `target_direcao` **sem limiar** e um `feature_set` menor (8 features). A esteira completa aqui usa 29 features (14 originais + 15 lags) — mais features = mais overfitting com 215 amostras de validação. O limiar resolve o overtrading mas não o overfitting.

> **A lição de engenharia**: preprocessamento reduz turnover e fees, mas não substitui edge. O modelo precisa de features com poder preditivo real, não apenas mais features bem processadas. A esteira completa é **necessária mas não suficiente** — ela é a infraestrutura que torna o deploy possível, mas o edge vem das features.

---

## Passo 5 — Componentes disponíveis na esteira

A esteira de ML do CT Lab tem 30+ componentes. Aqui está a referência completa:

### Preprocessamento (estimadores com fit no treino)

| Componente | Op | Função |
|---|---|---|
| `winsorize` | Aprende limites | Clipa outliers nos quantis q_inf/q_sup |
| `imputar` | Aprende média/mediana | Preenche NaN |
| `scaler_zscore` | Aprende μ/σ | Normaliza para z-score |
| `scaler_minmax` | Aprende min/max | Normaliza para [0, 1] |
| `scaler_robust` | Aprende mediana/IQR | Normaliza robusto a outliers |
| `scaler_maxabs` | Aprende max abs | Normaliza preservando sinal |
| `reduzir_pca` | Aprende componentes | Redução de dimensionalidade |
| `selecionar_correlacao` | Aprende top_k | Seleciona features por correlação |
| `selecionar_variancia` | Aprende limiar | Remove features de baixa variância |

### Transformações (sem fit)

| Componente | Op | Função |
|---|---|---|
| `gerar_lags` | Gera lags | Cria t-1, t-2, ... de features |
| `transformar_coluna` | Transforma | log, sqrt, abs, sinal, retorno_pct |
| `interagir_colunas` | Combina | razão, produto, diferença |
| `features_calendario` | Adiciona | hora_dia, dia_semana, sin/cos |
| `preencher_temporal` | Preenche | ffill ou bfill |

### Avaliação

| Componente | Op | Função |
|---|---|---|
| `avaliar` | Métricas classificação | acurácia, F1, precisão, revocação |
| `avaliar_regressao` | Métricas regressão | R², MSE, MAE |
| `avaliar_cv` | Validação cruzada | Métricas agregadas por fold |
| `avaliar_clustering` | Métricas clustering | silhouette, inertia |
| `avaliar_anomalia` | Métricas anomalia | precision@n, recall |

### Modelos

| Backend | Familia | Hiperparâmetros |
|---|---|---|
| Centroide | `centroide` | — |
| GBDT | `gbdt` | `max_depth`, `n_estimators`, `learning_rate` |
| MLP | `mlp` | `oculto`, `epocas`, `lr` |
| KMeans | `kmeans` | `n_clusters` |
| Isolation Forest | `isolation_forest` | `contamination` |
| Custom Python | `modelo_custom` | Script inline com `treinar`/`inferir` |

---

## Próximos passos

- **Modelo custom (LSTM)**: escreva um backend Python com LSTM usando `modelo_custom` — o componente aceita script inline com funções `treinar`/`inferir`. Veja o catálogo em `ct://ml/catalog`.
- **Otimizar hiperparâmetros**: use o nó `otimizar_hiperparametros` dentro da esteira para fazer grid search de `max_depth`, `n_estimators`, `learning_rate` automaticamente.
- **Fork da doutrina**: use a predição da esteira completa como gatilho do gestor adaptativo da lib `grupo` — o modelo diz a direção, o gestor gerencia a saída — veja [Fork da Doutrina](./11-fork-doutrina.pt.md).
- **Regime + esteira**: adicione ADX como feature (exemplo 09) para que o modelo aprenda a ajustar predições por regime.

---

> Voltar para: [README](../README.md) · [Regime + Modelo](./09-regime-modelo.pt.md) · [Modelo GBDT](./06-modelo-gbdt.pt.md) · [Fork da Doutrina](./11-fork-doutrina.pt.md)

_Last updated: 2026-08-12_
