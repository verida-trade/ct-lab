# 07 — Modelo MLP: Rede Neural com PyTorch

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Modelo GBDT](./06-modelo-gbdt.pt.md), [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md)

No [exemplo 06](./06-modelo-gbdt.pt.md), você treinou um GBDT que superou as taxas — +$12.447 de lucro líquido com fee de 0.1%. Mas o GBDT é um modelo de **árvores**: ele tabula o espaço de features em regiões retangulares. Será que uma **rede neural** — que pode aprender relações não-lineares suaves — faz melhor?

Neste exemplo, você vai treinar um **MLP** (Multi-Layer Perceptron) via PyTorch, usando as mesmas features e a mesma esteira de ML do exemplo 06. O objetivo é comparar:

- **GBDT** (árvores) vs **MLP** (rede neural)
- Qual modelo gera mais edge?
- Qual é mais robusto a taxas?
- Como cada modelo lida com confiança e exposição?

> **MLP** é uma rede neural feedforward: camada de entrada → camada oculta (com ativação ReLU) → camada de saída (softmax para classificação). Treina com gradient descent (Adam). O backend `mlp` do CT Lab usa PyTorch e é built-in — não precisa instalar nada extra.

---

## O problema

Você quer prever a direção do próximo candle de BTC, igual ao exemplo 06. Mas em vez de árvores, você quer testar uma rede neural. A motivação:

1. **Não-linearidade suave**: o MLP aprende fronteiras de decisão suaves, não degraus.
2. **Confiança calibrada**: o MLP produz probabilidades (softmax) — você pode filtrar trades por convicção.
3. **Menos overfitting com poucos dados**: o MLP tem menos parâmetros que um GBDT de 100 árvores.

A pergunta é: **um MLP simples (32 neurônios, 300 épocas) supera o GBDT no mesmo dataset?**

> O backend `mlp` do CT Lab já vem embutido. Ele usa `torch==2.12.0` (instalado automaticamente via `uv` no ambiente Python). O modelo é um `nn.Sequential(Linear → ReLU → Linear)` — a arquitetura mais simples de rede neural.

---

## Passo 1 — Reutilizar as features do exemplo 06

O exemplo 06 já construiu o pipeline de features em `ct://derived/btc_gbdt_features` (RSI, MACD, ADX, Bollinger, ATR, volume). O MLP usará **exatamente as mesmas features** — a comparação é justa.

```
As features já estão em ct://derived/btc_gbdt_features (8 colunas, 1712 linhas).
```

Se precisar reconstruir, rode o pipeline do [exemplo 06, Passo 1](./06-modelo-gbdt.pt.md).

---

## Passo 2 — Treinar o modelo MLP

A esteira é idêntica à do GBDT — só muda o `familia` no nó `modelo`:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_mlp_model",
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
        "componente": {
          "op": "modelo",
          "familia": "mlp",
          "tarefa": "classificacao",
          "hiperparametros": { "oculto": 32, "epocas": 300, "lr": 0.01 }
        },
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

### O que muda em relação ao GBDT

| Campo | GBDT | MLP |
|---|---|---|
| `familia` | `"gbdt"` | `"mlp"` |
| `hiperparametros` | `n_estimators`, `max_depth`, `learning_rate` | `oculto`, `epocas`, `lr` |
| Backend | scikit-learn | PyTorch |
| Dependência | `scikit-learn==1.9.0` | `torch==2.12.0` |

### Resultado do treino

```json
{
  "modelo_uri": "ct://models/btc_mlp_model",
  "predicao_uri": "ct://derived/btc_mlp_model_pred",
  "familia": "mlp",
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

Mesmas 20 features, mesmas 3 classes. A diferença está dentro do modelo: o MLP tem 32 neurônios na camada oculta (32×20 + 32×3 = 736 parâmetros), enquanto o GBDT tem 100 árvores de profundidade 5.

---

## Passo 3 — Aplicar o modelo e rodar backtest

### Aplicar

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "btc_mlp_model",
    "fonte": "ct://derived/btc_gbdt_features",
    "nome": "btc_mlp_pred"
  }
}
```

### Backtest (com taxas)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_mlp_pred",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "btc_mlp_bt_fee"
  }
}
```

### Resultado (com fee 0.1%)

```json
{
  "num_trades": 56,
  "pnl_total": -4056.30,
  "pnl_bruto": 3122.70,
  "fees_totais": 7179.00,
  "retorno_total": -0.406,
  "sharpe": -0.059,
  "sortino": -0.070,
  "win_rate": 0.268,
  "profit_factor": 0.424,
  "drawdown_max": 0.407,
  "num_wins": 15,
  "num_losses": 41,
  "avg_win": 199.0,
  "avg_loss": -171.7,
  "exposicao": 0.054
}
```

### Resultado (sem fee)

```json
{
  "num_trades": 56,
  "pnl_total": 3122.70,
  "pnl_bruto": 3122.70,
  "fees_totais": 0,
  "retorno_total": 0.312,
  "sharpe": 0.047,
  "sortino": 0.080,
  "win_rate": 0.607,
  "profit_factor": 1.983,
  "drawdown_max": 0.066,
  "exposicao": 0.054
}
```

---

## Passo 4 — Comparação: MLP vs GBDT

### Tabela comparativa (mesmas features, mesmos lags, mesmo walk-forward)

| Métrica | MLP (fee 0.1%) | GBDT (fee 0.1%) | MLP (sem fee) | GBDT (sem fee) |
|---|---|---|---|---|
| `pnl_total` | **−$4.056** | **+$12.447** | +$3.123 | +$60.442 |
| `pnl_bruto` | $3.123 | $60.442 | $3.123 | $60.442 |
| `num_trades` | 56 | 374 | 56 | 374 |
| `win_rate` | 27% | 39% | 61% | 98% |
| `profit_factor` | 0.42 | 2.19 | 1.98 | 104.7 |
| `exposicao` | 5.4% | 28.1% | 5.4% | 28.1% |
| `drawdown_max` | 40.7% | 9.4% | 6.6% | 0.7% |
| `sharpe` | −0.059 | 0.120 | 0.047 | 0.384 |

### A leitura

1. **GBDT supera as taxas; MLP não.** O GBDT gera $60k de PnL bruto — taxas de $48k ainda deixam $12k de lucro. O MLP gera apenas $3.1k de PnL bruto — as taxas de $7.2k devoram tudo e still sobra −$4k.

2. **O MLP é muito mais seletivo.** 56 trades vs 374 — exposição de 5.4% vs 28.1%. O MLP só opera quando tem convicção (classe +1 ou -1), e fica zerado na maioria das barras (classe 0). O GBDT opera muito mais porque classifica mais barras como +1 ou -1.

3. **Menos trades = menos edge bruto.** Com apenas 56 trades, o MLP acumula $3.1k de PnL bruto. Mesmo com menos taxas ($7.2k vs $48k do GBDT), o PnL bruto é tão baixo que não cobre nem as taxas reduzidas.

4. **Win rate sem taxas: MLP 61% vs GBDT 98%.** O GBDT quase nunca erra direção quando prevê +1 ou -1. O MLP erra 39% das vezes. A vantagem do GBDT é a robustez direcional — poda iterativamente os erros.

> A lição não é "MLP é pior que GBDT". É que **com este dataset e esta configuração**, o GBDT extrai mais edge. O MLP é um modelo mais simples (736 parâmetros vs centenas de nós de árvore) — pode ser que com mais dados, normalização de features, ou uma arquitetura mais rica (LSTM, Transformer), a rede neural supere. Mas com 1712 barras e 20 features, o GBDT ganha.

---

## Passo 5 — Filtro de confiança com probabilidades

O MLP produz probabilidades via softmax. Em vez de operar sempre que `pred ≠ 0`, podemos filtrar por convicção:

### Aplicar com probabilidades

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "btc_mlp_model",
    "fonte": "ct://derived/btc_gbdt_features",
    "nome": "btc_mlp_probas",
    "probas": true
  }
}
```

> Materializa 3 colunas: `p_0` (probabilidade de ficar), `p_1` (subir), `p_m1` (descer).

### Backtest com filtro de confiança

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_mlp_probas",
    "estrategia_script": "if ind[\"p_1\"][0] > 0.6 { comprado(1.0) } else if ind[\"p_m1\"][0] > 0.6 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "btc_mlp_confidence"
  }
}
```

Só opera quando a probabilidade de uma classe excede 60% — muito mais conservador.

### Resultado (filtro 60%)

```json
{
  "num_trades": 9,
  "pnl_total": -803.09,
  "pnl_bruto": 353.25,
  "fees_totais": 1156.34,
  "win_rate": 0.333,
  "profit_factor": 0.253,
  "exposicao": 0.005
}
```

### A leitura

O filtro de confiança reduz de 56 para **9 trades** — mas o resultado piora: PnL cai de −$4.056 para −$803, mas o PnL bruto cai mais ainda ($3.123 → $353). O filtro remove trades bons e ruins em proporção similar — a confiança do softmax do MLP não é discriminativa o suficiente.

> O filtro de confiança funciona melhor quando o modelo está **bem calibrado** (prob = frequência real). Redes neurais costumam ser mal calibradas — produzem Distribuições overconfident ou underconfident. O GBDT, por outro lado, não produz probabilidades confiáveis por padrão (as "probabilidades" do sklearn são suavizadas).

---

## Passo 6 — Anatomia do backend MLP

O backend `mlp` é embutido no servidor MCP e roda via PyTorch no ambiente `uv`. A arquitetura é:

```
                    ┌────────────────────────────────────┐
                    │        Backend MLP (PyTorch)       │
                    ├────────────────────────────────────┤
                    │                                    │
                    │  Entrada: X (N x 20)              │
                    │           │                        │
                    │           ▼                        │
                    │  ┌──────────────┐                  │
                    │  │ nn.Linear(20,│  pesos + bias    │
                    │  │       32)    │  (640 + 32)     │
                    │  └──────┬───────┘                  │
                    │         │                          │
                    │         ▼                          │
                    │  ┌──────────────┐                  │
                    │  │  nn.ReLU()   │  ativação        │
                    │  └──────┬───────┘                  │
                    │         │                          │
                    │         ▼                          │
                    │  ┌──────────────┐                  │
                    │  │ nn.Linear(32,│  pesos + bias    │
                    │  │        3)    │  (96 + 3)       │
                    │  └──────┬───────┘                  │
                    │         │                          │
                    │         ▼                          │
                    │  CrossEntropyLoss (classificação)  │
                    │  Adam optimizer (lr=0.01)          │
                    │  300 épocas                        │
                    │                                    │
                    │  Total: 739 parâmetros             │
                    │  (vs GBDT: 100 árvores × ~32 nós)  │
                    └────────────────────────────────────┘
```

### Hiperparâmetros do MLP

| Parâmetro | Default | Efeito |
|---|---|---|
| `oculto` | 16 | Nº de neurônios na camada oculta. Mais = mais capacidade, mais overfitting |
| `epocas` | 200 | Nº de passos de gradient descent. Mais = melhor fit no treino, risco de overfit |
| `lr` | 0.01 | Learning rate do Adam. Alto = converge rápido, risco de oscilação |
| `random_state` | — | Seed do torch (não controlado pelo backend MLP) |

> Para ajustar esses hiperparâmetros, use `otimizar_hiperparametros` igual ao exemplo 06 — mude `familia` para `"mlp"`.

---

## GBDT vs MLP: quando usar cada um?

| Critério | GBDT | MLP |
|---|---|---|
| **Dataset pequeno** (< 5k) | ✓ Excelente | ✗ Risk of overfit |
| **Dataset grande** (> 50k) | ✓ Bom (mas lento) | ✓ Bom |
| **Features categóricas** | ✓ Nativo | ✗ Precisa embedding |
| **Relações suaves** | ✗ Fronteiras degrau | ✓ Fronteiras suaves |
| **Interpretabilidade** | ✓ Feature importance | ✗ Caixa preta |
| **Calibração** | ✗ Não calibrado | ✗ Não calibrado (melhor que GBDT) |
| **Treino rápido** | ✓ Segundos | ✓ Segundos (MLP simples) |
| **Seriabilidade temporal** | ✗ Sem memória | ✗ Sem memória |
| **Turnover** | Alto (muitos trades) | Baixo (seletivo) |

> **Nem GBDT nem MLP têm memória temporal.** Ambos veem uma barra por vez, sem contexto das barras anteriores. Os lags (lag1, lag2, lag3) são um "patch" — dão ao modelo uma janela, mas limitada. Um **LSTM** (Long Short-Term Memory) tem memória interna e pode capturar dependências de longo prazo — é o próximo passo natural.

---

## Próximos passos

- **Combinar MLP + gestor adaptativo**: use a predição do MLP como gatilho da lib `grupo` — entra apenas quando o modelo prevê direção, deixa o gestor gerenciar a saída — veja [Fork da Doutrina](./11-fork-doutrina.pt.md).
- **Modelo custom (LSTM)**: escreva um backend Python com LSTM usando `modelo_custom` — veja [Esteira ML Completa](./10-esteira-ml-completa.pt.md).
- **Otimizar hiperparâmetros do MLP**: rode grid search variando `oculto`, `epocas`, `lr` — veja [Modelo GBDT, Passo 5](./06-modelo-gbdt.pt.md).

---

> Voltar para: [README](../README.md) · [Modelo GBDT](./06-modelo-gbdt.pt.md) · [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md)

_Last updated: 2026-08-12_
