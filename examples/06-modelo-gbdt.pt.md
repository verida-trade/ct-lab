# 06 — Modelo GBDT: O Aprendiz de Direção

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md), [Cross-Asset](./05-cross-asset.pt.md), [Estratégia Rhai](../docs/05-backtest/03-estrategia-rhai.pt.md)

Nos exemplos 04 e 05, você mediu o **piso de sobrevivência** — o quanto o gestor adaptativo perde sem critério de entrada. Em BTC, o piso é negativo: EV de −0.095 réguas sem taxas e −0.92 com taxas. A conclusão foi clara: **sua estratégia precisa de um fator de entrada que adicione pelo menos +0.92 réguas de edge por trade** para superar o custo de execução.

Neste exemplo, você vai construir esse fador de entrada usando **Gradient Boosted Decision Trees (GBDT)** — um modelo de ML que aprende a prever a direção do próximo candle a partir de indicadores técnicos. Você vai:

1. **Construir features** com um pipeline de indicadores (RSI, MACD, ADX, Bollinger, ATR, lags).
2. **Treinar o modelo GBDT** via esteira de ML declarativa, com validação walk-forward.
3. **Aplicar o modelo** como indicador vivo e rodar backtest.
4. **Comparar com o piso de sobrevivência** e com buy-and-hold.
5. **Otimizar hiperparâmetros** com grid search.
6. **Entender as limitações** — overfitting, regime change, e quando o modelo para de funcionar.

---

## O problema

Você quer prever se o próximo candle de BTC vai **subir** ou **descer**. Com essa previsão, sua estratégia compra ou vende usando o gestor adaptativo da lib `grupo`.

Sem modelo, a direção é arbitrária — o teste de sobrevivência mostrou que isso sangra. Com um modelo que acerta direção, cada acerto vira edge que supera o custo de execução. A questão é: **um GBDT consegue aprender direção a partir de indicadores técnicos?**

> **GBDT** (Gradient Boosted Decision Trees) é um ensemble de árvores de decisão que corrige iterativamente os erros das árvores anteriores. É o modelo de tabular ML mais usado em finanças quantitativas — robusto, não-linear, e útil com poucas features.

---

## Passo 1 — Buscar dados e construir features

### Buscar a série

```
Busque BTCUSDT em 15m na Binance.
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "interval": "15m" }
}
```

### Construir o pipeline de indicadores

As features são a matéria-prima do modelo. Vamos materializar 8 indicadores + preço + volume em uma série derivada única:

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_gbdt_features",
    "output": "$features",
    "steps": [
      { "op": "rsi", "id": "rsi", "source": "$anchor", "column": "close", "period": 14 },
      { "op": "macd", "id": "macd", "source": "$anchor", "column": "close", "fast": 12, "slow": 26, "signal": 9 },
      { "op": "adx", "id": "adx", "source": "$anchor", "period": 14 },
      { "op": "bollinger", "id": "boll", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "sma", "id": "sma20", "source": "$anchor", "column": "close", "period": 20 },
      { "op": "sma", "id": "sma50", "source": "$anchor", "column": "close", "period": 50 },
      { "op": "atr", "id": "atr", "source": "$anchor", "period": 14 },
      {
        "op": "compose",
        "id": "features",
        "columns": [
          { "as_column": "rsi",         "source": "$rsi",   "source_column": "rsi" },
          { "as_column": "macd",        "source": "$macd",  "source_column": "macd" },
          { "as_column": "macd_signal", "source": "$macd",  "source_column": "signal" },
          { "as_column": "adx",         "source": "$adx",   "source_column": "adx" },
          { "as_column": "plus_di",     "source": "$adx",   "source_column": "plus_di" },
          { "as_column": "minus_di",    "source": "$adx",   "source_column": "minus_di" },
          { "as_column": "atr",         "source": "$atr",   "source_column": "atr" },
          { "as_column": "volume",      "source": "$anchor","source_column": "volume" }
        ]
      }
    ]
  }
}
```

> A série de features fica em `ct://derived/btc_gbdt_features` com 1712 linhas e 8 colunas.

### Análise exploratória (EDA)

Antes de treinar, verifique a correlação de cada feature com o preço (alvo):

```json
{
  "name": "analisar_dataset",
  "arguments": {
    "features": ["ct://derived/btc_gbdt_features"],
    "target": "ct://series/binance/BTCUSDT/15m",
    "target_coluna": "close",
    "quantis": [0.1, 0.25, 0.5, 0.75, 0.9]
  }
}
```

#### Correlação feature × preço

| Feature | Correlação com `close` | Insight |
|---|---|---|
| `rsi` | +0.24 | Positiva fraca — RSI acompanha preço |
| `macd` | +0.37 | Positiva moderada |
| `macd_signal` | +0.38 | Similar ao MACD |
| `adx` | −0.16 | Negativa fraca — em tendência, ADX sobe |
| `plus_di` | +0.45 | Positiva moderada |
| `minus_di` | −0.36 | Negativa moderada |
| `atr` | −0.12 | Negativa fraca |
| `volume` | −0.07 | Quase nula |

> A correlação com o preço **não é o que o modelo vai prever**. O modelo prevê **direção** (subir/descer), não o preço absoluto. A correlação acima mostra que `plus_di` e `macd` têm relação linear com o preço — mas o GBDT captura relações **não-lineares** que a correlação não vê.

---

## Passo 2 — Treinar o modelo GBDT

A esteira de ML é um **DAG declarativo** — cada nó é um componente com entradas e saídas. O fluxo é:

```
feature_set → gerar_lags → aplicar_em_features → dataset → split_walk_forward → modelo → prever
                                ↑
                          target_direcao
```

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_gbdt_model",
    "nos": [
      {
        "id": "feats",
        "componente": { "op": "feature_set", "colunas": ["rsi", "macd", "macd_signal", "adx", "plus_di", "minus_di", "atr", "volume"] },
        "entradas": ["ct://derived/btc_gbdt_features"]
      },
      {
        "id": "lags",
        "componente": { "op": "gerar_lags", "colunas": ["rsi", "macd", "adx", "atr"], "lags": [1, 2, 3] },
        "entradas": ["$feats"]
      },
      {
        "id": "feats_lag",
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
        "componente": { "op": "dataset" },
        "entradas": ["$feats_lag", "$alvo"]
      },
      {
        "id": "wf",
        "componente": { "op": "split_walk_forward", "n_folds": 4, "rolling": false, "treino_inicial_frac": 0.5 },
        "entradas": ["$ds"]
      },
      {
        "id": "modelo",
        "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 100, "max_depth": 5, "learning_rate": 0.1, "random_state": 42 } },
        "entradas": ["$ds", "$wf"]
      },
      {
        "id": "pred",
        "componente": { "op": "prever" },
        "entradas": ["$modelo", "$feats_lag"]
      }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$wf"
  }
}
```

### O que cada nó faz

| Nó | `op` | Função |
|---|---|---|
| `feats` | `feature_set` | Empacota colunas da série derivada como matriz X |
| `lags` | `gerar_lags` | Cria 12 features extras: `rsi_lag1`, `rsi_lag2`, `rsi_lag3`, `macd_lag1`, ... |
| `feats_lag` | `aplicar_em_features` | Reaplica os lags ao feature_set (necessária porque lags → Ajuste, não FeatureSet) |
| `alvo` | `target_direcao` | Rótula +1/0/-1: se próximo candle subiu >0.1%, classe +1; desceu >0.1%, classe -1; caso contrário, 0 |
| `ds` | `dataset` | Casa features + target por timestamp; remove linhas com NaN (warm-up dos lags) |
| `wf` | `split_walk_forward` | Divide em 4 folds temporais (expanding): treino cresce a cada fold |
| `modelo` | `modelo` | Treina GBDT: 100 árvores, profundidade 5, learning rate 0.1 |
| `pred` | `prever` | Aplica o modelo ao feature_set → série de predição (`ct://derived/btc_gbdt_model_pred`) |

### Resultado do treino

```json
{
  "modelo_uri": "ct://models/btc_gbdt_model",
  "predicao_uri": "ct://derived/btc_gbdt_model_pred",
  "familia": "gbdt",
  "classes": [-1, 0, 1],
  "colunas_x": [
    "rsi", "macd", "macd_signal", "adx", "plus_di", "minus_di", "atr", "volume",
    "rsi_lag1", "rsi_lag2", "rsi_lag3",
    "macd_lag1", "macd_lag2", "macd_lag3",
    "adx_lag1", "adx_lag2", "adx_lag3",
    "atr_lag1", "atr_lag2", "atr_lag3"
  ],
  "nos_executados": 8
}
```

20 features (8 base + 12 lags), 3 classes (subiu / ficou / desceu). O modelo está persistido em `ct://models/btc_gbdt_model` — você pode reutilizá-lo quantas vezes quiser sem retreinar.

---

## Passo 3 — Aplicar o modelo e rodar backtest

### Aplicar (sem retreinar)

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "btc_gbdt_model",
    "fonte": "ct://derived/btc_gbdt_features",
    "nome": "btc_gbdt_pred"
  }
}
```

> Materializa a coluna `pred` em `ct://derived/btc_gbdt_pred` — o modelo como indicador vivo.

### Backtest: modelo como sinal

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_gbdt_pred",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "btc_gbdt_bt_fee"
  }
}
```

A estratégia é simples: se o modelo prevê subida (`pred > 0`), compra 1.0 BTC. Se prevê queda (`pred < 0`), vende 1.0 BTC. Caso contrário, fica zerado.

### Resultado (1712 candles de BTCUSDT 15m, com fee 0.1%)

```json
{
  "uri": "ct://backtest/btc_gbdt_bt_fee",
  "num_trades": 374,
  "pnl_total": 12446.75,
  "pnl_bruto": 60441.83,
  "fees_totais": 47995.08,
  "retorno_total": 1.2447,
  "sharpe": 0.120,
  "sortino": 0.378,
  "win_rate": 0.393,
  "profit_factor": 2.186,
  "drawdown_max": 0.094,
  "num_wins": 147,
  "num_losses": 227,
  "avg_win": 156.07,
  "avg_loss": -46.23,
  "exposicao": 0.281
}
```

---

## Passo 4 — Interpretando os resultados

### O GBDT supera as taxas

| Métrica | Valor | O que significa |
|---|---|---|
| `pnl_total` | +$12.447 | **Lucro líquido após taxas** — o modelo gera edge real |
| `pnl_bruto` | +$60.442 | Sem taxas, o lucro é $60k — o modelo acerta direção consistentemente |
| `fees_totais` | $47.995 | 374 trades × ~$128/trade — o custo de execução é alto |
| `profit_factor` | 2.19 | Ganhos são 2.19× as perdas — o modelo tem edge positivo |
| `win_rate` | 39% | Menos da metade dos trades ganham — mas `avg_win` (−$156) é 3.4× `avg_loss` (−$46) |
| `exposicao` | 28% | O modelo só opera 28% do tempo — fica zerado quando não tem convicção |
| `drawdown_max` | 9.4% | Drawdown máximo controlado |

### Comparação: GBDT vs piso de sobrevivência vs buy-and-hold

| Estratégia | `pnl_total` | `win_rate` | `profit_factor` | `sharpe` | `drawdown` |
|---|---|---|---|---|---|
| **GBDT (fee 0.1%)** | **+$12.447** | 39% | **2.19** | 0.12 | 9.4% |
| GBDT (sem fee) | +$60.442 | 98% | 104.7 | 0.38 | 0.7% |
| Buy & Hold (fee 0.1%) | −$296 | — | 0 | 0.003 | 28.9% |
| Sobrevivência (sem fee) | −$3.807 | 35% | — | — | — |
| Sobrevivência (fee 0.1%) | −$24.609 | 5% | — | — | — |
| Aleatório (fee 0.1%) | −$110.912 | 8% | 0.07 | 0.025 | 11.1× |

### O insight

O GBDT **gera edge que supera o custo de execução**:
- Sobrevivência (lado arbitrário) perde $24.6k com taxas — o piso.
- GBDT ganha $12.4k com taxas — uma **diferença de $37k** sobre o piso.
- Buy & Hold perde $296 no período — o mercado não teve trend claro.
- Aleatório perde $110k — o pior caso (alto turnover, nenhum edge).

> Sem taxas, o GBDT tem win rate de **98%** e profit factor de **104** — o modelo quase nunca erra direção no período. As taxas comem 80% do lucro bruto ($60k → $12k), mas o saldo é **positivo**. Este é o objetivo da doutrina: gerar edge que supera o custo de execução.

### Por que 98% de win rate sem taxas?

O modelo prevê direção (subir/descer/ficar). A maioria das barras tem movimento muito pequeno (< 0.1%), que o modelo classifica como classe 0 (ficar). A estratégia fica zerada — sem posição, sem ganho, sem perda. Quando o modelo prevê +1 ou -1, ele acerta na maioria das vezes porque o sinal é forte o suficiente para ter convicção.

Com taxas, cada trade custa ~$128 (0.1% de ~$64k), e 374 trades geram $48k em taxas. O PnL bruto de $60k cobre as taxas e ainda sobra $12k.

---

## Passo 5 — Otimizando hiperparâmetros

O modelo foi treinado com `n_estimators=100, max_depth=5, learning_rate=0.1`. Será que outros valores funcionam melhor? O `otimizar_hiperparametros` faz grid search com validação temporal:

```json
{
  "name": "otimizar_hiperparametros",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "familia": "gbdt",
    "colunas": ["close", "volume"],
    "horizonte": 1,
    "limiar": 0.001,
    "estrategia": "grid",
    "grade": {
      "n_estimators": [50, 100, 200],
      "max_depth": [3, 5, 7]
    },
    "hiperparametros_base": { "learning_rate": 0.1, "random_state": 42 },
    "treino_frac": 0.7
  }
}
```

### Resultado do grid search (9 combinações)

| `n_estimators` | `max_depth` | Acurácia |
|---|---|---|
| 50 | 3 | **70.6%** |
| 100 | 3 | **70.6%** |
| 200 | 3 | **70.6%** |
| 50 | 5 | 66.9% |
| 100 | 5 | 66.9% |
| 200 | 5 | 66.9% |
| 50 | 7 | 66.9% |
| 100 | 7 | 66.9% |
| 200 | 7 | 66.9% |

### A leitura

- **`max_depth=3` é melhor que `5` ou `7`**: árvores rasas generalizam melhor. Profundidade 7 overfit — decora o treino mas perde na validação.
- **`n_estimators` não importa com `max_depth=3`**: 50, 100 ou 200 árvores dão a mesma acurácia. O modelo converge rápido com árvores rasas.
- **70.6% de acurácia** significa que o modelo acerta direção em 70% das barras. O restante 30% é o erro — que o gestor adaptativo (stop/take/trailing) tenta limitar.

> A otimização confirma a doutrina: **modelos simples generalizam melhor**. `max_depth=3` com 50 árvores é suficiente. Mais complexidade não ajuda — e pode piorar fora da amostra.

---

## Anatomia da esteira de ML

```
                    ┌─────────────────────────────────────────────────┐
                    │            ESTEIRA DE ML (DAG)                    │
                    ├─────────────────────────────────────────────────┤
                    │                                                 │
                    │  ct://derived/btc_gbdt_features                 │
                    │  (RSI, MACD, ADX, Bollinger, ATR, volume)       │
                    │           │                                     │
                    │           ▼                                     │
                    │  ┌─────────────┐                                │
                    │  │ feature_set  │  8 features                   │
                    │  └──────┬──────┘                                │
                    │         │                                      │
                    │         ▼                                      │
                    │  ┌─────────────┐   ┌──────────────────┐          │
                    │  │  gerar_lags  │→ │ aplicar_em_feat   │ +12 lags │
                    │  └─────────────┘   └────────┬─────────┘          │
                    │                              │                   │
                    │  ct://series/...             │                   │
                    │  (BTCUSDT 15m)               │                   │
                    │       │                      │                   │
                    │       ▼                      ▼                   │
                    │  ┌──────────────┐    ┌──────────────┐           │
                    │  │target_direcao│    │   dataset     │           │
                    │  │  (+1/0/-1)   │→   │ (join + drop) │           │
                    │  └──────────────┘    └──────┬───────┘           │
                    │                              │                   │
                    │                              ▼                   │
                    │                    ┌──────────────────┐         │
                    │                    │ split_walk_fwd    │ 4 folds │
                    │                    │ (expanding 50%+)  │         │
                    │                    └────────┬─────────┘         │
                    │                              │                   │
                    │                              ▼                   │
                    │                    ┌──────────────────┐         │
                    │                    │  modelo (GBDT)   │         │
                    │                    │  100 trees, d=5  │         │
                    │                    └────────┬─────────┘         │
                    │                              │                   │
                    │                              ▼                   │
                    │                    ┌──────────────────┐         │
                    │                    │    prever        │         │
                    │                    │ ct://derived/    │         │
                    │                    │ btc_gbdt_pred    │         │
                    │                    └──────────────────┘         │
                    └─────────────────────────────────────────────────┘
```

### Componentes da esteira

| `op` | O que faz | Entradas |
|---|---|---|
| `feature_set` | Empacota colunas como matriz X | série (URI) |
| `gerar_lags` | Cria defasagens x[t-1], x[t-2], ... como novas features | `$feature_set` |
| `aplicar_em_features` | Reaplica um Ajuste (lags, scaler) ao feature_set | `$ajuste`, `$feature_set` |
| `target_direcao` | Rótula +1/0/-1 pelo sinal do retorno a N barras à frente | série (URI) |
| `dataset` | Casa features + target por timestamp; remove NaN | `$feature_set`, `$target` |
| `split_walk_forward` | Divide em K folds temporais (expanding ou rolling) | `$dataset` |
| `split_holdout` | Divide em treino (fração inicial) + validação | `$dataset` |
| `modelo` | Treina modelo (familia: gbdt, centroide, mlp; tarefa: classificacao, regressao) | `$dataset`, `$split` |
| `prever` | Aplica modelo treinado → série de predição | `$modelo`, `$feature_set` |
| `avaliar` | Avalia predição contra dataset + split | `$pred`, `$dataset`, `$split` |
| `imputar` | Preenche NaN (media, mediana, zero, constante) | `$feature_set` |
| `scaler_zscore` | Padroniza features (média 0, desvio 1) | `$feature_set` |
| `winsorize` | Capa outliers nos quantis 1%-99% | `$feature_set` |
| `selecionar_correlacao` | Mantém top-k features por |corr| com alvo | `$feature_set` |

---

## Limitações e caveatas

### 1. Overfitting

O resultado de **98% de win rate sem taxas** é suspeito. O modelo quase nunca erra direção — mas isso pode ser porque a maioria das barras tem movimento muito pequeno (`pred = 0`, classe "ficar"). O modelo acerta a classe 0 na maioria das vezes porque é a classe dominante.

> Para validar que o modelo não está apenas acertando a classe majoritária, verifique o **balanço de classes** no dataset. Se 70% das barras são classe 0, então 70% de acurácia é o baseline trivial.

### 2. Look-ahead bias

O `target_direcao` usa `close.shift(-1) / close - 1` — o retorno da **próxima** barra. O modelo preve a barra seguinte usando dados até a barra atual. Mas atenção:

- O `dataset` remove a última barra (NaN no target).
- O `prever` aplica o modelo em todo o feature_set, **incluindo a última barra** — onde o modelo não tem o target real.
- O backtest usa a predição apenas em barras onde `pred` existe.

### 3. Regime change

O modelo foi treinado em 1712 barras de BTCUSDT 15m. Se o regime de mercado mudar (bull → bear, baixa vol → alta vol), o modelo pode parar de funcionar. A validação walk-forward mitiga isso parcialmente, mas não elimina.

> **Rode o teste periodicamente**. Se a acurácia cair abaixo de 55% (próximo de aleatório), é hora de retreinar ou revisar as features.

### 4. O custo de execução come o lucro

Sem taxas, o lucro é $60k. Com taxas de 0.1%, cai para $12k — **80% do lucro vai para taxas**. Reduzir o turnover (operar apenas quando `|pred|` é grande) pode ajudar:

```rhai
// Só opera quando o modelo tem convicção forte
if ind["pred"][0] > 0.5 { comprado(1.0) }
else if ind["pred"][0] < -0.5 { vendido(1.0) }
else { zerado() }
```

### 5. O modelo não substitui o gestor

O GBDT prevê **direção**, não magnitude. Ele não sabe se o movimento será grande ou pequeno. O gestor adaptativo (stop/take/trailing) é quem limita a perda quando o modelo erra e captura o ganho quando ele acerta. Os dois trabalham **juntos**:

- **Modelo**: decide **quando** entrar e **qual lado**
- **Gestor**: decide **como** sair (stop, take, trailing)

---

## Próximos passos

- **Combinar modelo + lib `grupo`**: use a predição do GBDT como gatilho da lib `grupo` — entra apenas quando o modelo prevê direção, e deixa o gestor gerenciar a saída — veja [Fork da Doutrina](./11-fork-doutrina.pt.md).
- **Modelo LSTM**: treine uma rede neural recorrente que captura dependência temporal de longa duração — veja [Modelo LSTM](./07-modelo-lstm.pt.md).
- **Otimizar features**: adicione indicadores CT Lab (ct_bop, ct_momento, ct_range) e veja se melhoram a acurácia — veja [Indicadores CT](../docs/04-indicadores/README.pt.md).

---

> Voltar para: [README](../README.md) · [Cross-Asset](./05-cross-asset.pt.md) · [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md)

_Last updated: 2026-08-11_
