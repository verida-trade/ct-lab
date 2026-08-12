# Receita 09 — Regime + Modelo Preditivo

> **Nível:** Avançado · **Premium:** Sim · **Tempo:** ~15 min

**Pré-requisitos:** Receitas 06 (esteira ML) e 08 (backtests e teste de sobrevivência).

---

## O que é regime?

O regime é o estado de mercado em um dado momento — tendência ativa, direção,
fase do ciclo e progresso. O CT Lab detecta regime via `ct_tendencia`, que
transforma OHLCV bruto em cinco colunas:

- **`tend_ativa`** — 1 se há tendência estatisticamente significativa, 0 caso contrário
- **`direcao`** — +1 (alta), −1 (baixa) ou 0 (lateral)
- **`fase`** — etapa do ciclo (formação / confirmação / exaustão)
- **`progresso`** — quão avançada está a tendência (0 a 1)
- **`nivel_rompido`** — preço de rompimento que validou a tendência

> **Conceito-chave:** Regime é *setup*, não fundação. O modelo precisa bater o
> piso de lado aleatório no mesmo teste de sobrevivência.

---

## Passo 1 — Buscar série e criar ct_tendencia

```python
# 1-A: buscar candles 15m do BTCUSDT
buscar_binance(symbol="BTCUSDT", interval="15m", limit=2000)
#  → ct://series/binance/BTCUSDT/15m

# 1-B: detectar regime com parâmetros validados
ct_tendencia(
    uri="ct://series/binance/BTCUSDT/15m",
    janela_zscore=96,
    janela_atr=14,
    k_mov=3,
    z_min=2.0,
    tau_r=1.5,
)
#  → ct://derived/ct_tendencia_binance_BTCUSDT_15m
#    colunas: tend_ativa, direcao, fase, progresso, nivel_rompido
```

| Parâmetro       | Valor | Função                                     |
|-----------------|-------|--------------------------------------------|
| `janela_zscore` | 96    | Janela para normalização (z-score)         |
| `janela_atr`    | 14    | Janela do ATR para normalizar movimentos   |
| `k_mov`         | 3     | Multiplicador de movimento mínimo          |
| `z_min`         | 2.0   | Z-score mínimo para confirmar tendência    |
| `tau_r`         | 1.5   | Persistência mínima (em barras)            |

---

## Passo 2 — Materializar features RSI

```python
materializar_indicador(
    uri="ct://series/binance/BTCUSDT/15m",
    indicador="rsi",
    periodo=14,
    destino="ct://derived/btc_rsi_14",
)
#  → ct://derived/btc_rsi_14  (coluna: rsi)
```

---

## Passo 3 — Montar esteira ML

A esteira combina features de regime (`direcao`, `fase`, `progresso`) com RSI e
gera lags para dar contexto temporal ao modelo.

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_regime_model",
    "nos": [
      { "id": "feats", "componente": { "op": "feature_set", "colunas": ["direcao", "fase", "progresso", "rsi"] }, "entradas": ["ct://derived/ct_tendencia_binance_BTCUSDT_15m", "ct://derived/btc_rsi_14"] },
      { "id": "lags", "componente": { "op": "gerar_lags", "colunas": ["direcao", "fase", "progresso", "rsi"], "lags": [1, 2, 3, 5] }, "entradas": ["$feats"] },
      { "id": "feats_lag", "componente": { "op": "aplicar_em_features" }, "entradas": ["$lags", "$feats"] },
      { "id": "alvo", "componente": { "op": "target_direcao", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "ds", "componente": { "op": "dataset" }, "entradas": ["$feats_lag", "$alvo"] },
      { "id": "wf", "componente": { "op": "split_walk_forward", "n_folds": 4, "rolling": false, "treino_inicial_frac": 0.5 }, "entradas": ["$ds"] },
      { "id": "modelo", "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 150, "max_depth": 5 } }, "entradas": ["$ds", "$wf"] },
      { "id": "pred", "componente": { "op": "prever" }, "entradas": ["$modelo", "$feats_lag"] }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$wf",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

Resultado: `ct://derived/btc_regime_model_pred`

### O que cada nó faz

| Nó          | Op                   | Função                                                             |
|-------------|----------------------|--------------------------------------------------------------------|
| `feats`     | `feature_set`        | Seleciona colunas de regime + RSI das URIs de origem               |
| `lags`      | `gerar_lags`         | Cria Defasagens (1, 2, 3, 5) das 4 colunas → contexto temporal     |
| `feats_lag` | `aplicar_em_features`| Funde features originais com lags em um único DataFrame            |
| `alvo`      | `target_direcao`     | Cria alvo binário: direção do preço em 5 barras (acima/abaixo)     |
| `ds`        | `dataset`            | Monta dataset (X = feats+lags, y = alvo)                           |
| `wf`        | `split_walk_forward` | Divide em 4 folds walk-forward (50% treino inicial)               |
| `modelo`    | `modelo`             | Treina GBDT (150 árvores, depth 5) em cada fold                     |
| `pred`      | `prever`             | Gera previsões ponto-a-ponto usando o modelo treinado             |

---

## Passo 4 — Backtest da predição

A predição do modelo é materializada como série derivada e alimentada no
backtest como um indicador de sinal.

```python
ct_backtest(
    uri="ct://derived/btc_regime_model_pred",
    capital_inicial=1000,
    fee_pct=0.001,
    survival=True,
)
```

Compare o resultado com o **piso de lado aleatório** (Receita 08) no mesmo
período. Se o modelo não bate o piso, as features de regime não são suficientes
— volte aos Passos 1-3.

---

## Interpretação

- **Features de regime** dão contexto ao modelo: tendência ativa, direção e
  fase ajudam o GBDT a distinguir momentos de momentum de lateralização.
- **Lags** capturam dinâmica: o regime há 3 barras atrás importa tanto quanto
  o regime atual para prever a próxima direção.
- **Walk-forward** evita viés de look-ahead — cada fold treina só com dados
  anteriores ao teste.
- **O modelo precisa bater o piso de sobrevivência.** Regime melhora o sinal,
  mas não garante vantagem estatística se a série é eficiente demais.
- **Regime é setup, não fundação.** A fundação é a bateria de testes
  (survival, random-side). O regime apenas dá um contexto melhor ao modelo.

---

## Variações

- **`janela_zscore=48`** — regime mais reativo, detecta tendências curtas;
  aumenta ruído, exigir `z_min` mais alto (2.5).
- **Adicionar `ct_range` features** — incluir amplitude e posição dentro do
  canal para capturar squeezes e dilatações.
- **Regressão em vez de classificação** — usar `target_retorno` (log-retorno
  em N barras) para prever magnitude, não só direção.
- **Aumentar horizonte** — `target_direcao` com `horizonte=10` ou `15` para
  capturar movimentos mais longos; requer mais lags (7, 10) para alinhar
  contexto.
