# Receita 06 — Treinar Modelo de Direção (GBDT)

**Nível:** Avançado · **Plano:** Premium · **Pré-requisitos:** Receita 01 (Séries), Receita 02 (Backtest), Receita 04 (Indicadores)

Esta receita constrói um pipeline completo de machine learning usando Gradient Boosted Decision Trees (GBDT) para prever a direção do preço em janelas de 5 barras, com backtest da predição.

---

## O pipeline ML

O fluxo do CT Lab para modelagem preditiva é linear e declarativo:

- **Feature set** → seleciona colunas de um indicador materializado como features
- **Lags** → gera versões defasadas (t-1, t-2, t-3) para dar memória ao modelo
- **Target** → cria o alvo de direção (alta/baixa/neutro) num horizonte definido
- **Split** → divide o dataset em folds walk-forward para validação fora da amostra
- **Modelo** → treina o GBDT com hiperparâmetros configuráveis
- **Prever** → gera predições que podem ser usadas como indicador no backtest

---

## Passo 1 — Buscar série

```
buscar_binance(symbol="BTCUSDT", interval="15m")
```

Resultado: 1 724 candles cacheados em `ct://series/binance/BTCUSDT/15m`.

---

## Passo 2 — Materializar features

Usamos `materializar_indicador` para criar indicadores que servem de features. A receita é um mapa Rhai com expressões válidas:

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_ml_feats",
    "receita": "#{\"rsi\": rsi(close, 14), \"atr\": atr(high, low, close, 14)}"
  }
}
```

**Resultado:**

| Campo        | Valor                          |
|--------------|--------------------------------|
| uri          | `ct://derived/btc_ml_feats`    |
| row_count    | 1 724                          |
| value_names  | `["atr", "rsi"]`              |

> **Importante:** Use funções Rhai suportadas (`rsi`, `atr`, `adx`, `macd`). Para `macd`, passe os 3 períodos: `macd(close, 12, 26, 9)["hist"]`. Evite indexadores como `close[5]` (indisponível). Para `adx`, múltiplas colunas são retornadas.

---

## Passo 3 — Montar esteira ML

A função `montar_esteira_ml` recebe um array de nós, cada um com `id`, `componente` (com campo `op`) e `entradas`:

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_gbdt_dir5",
    "nos": [
      { "id": "feats",      "componente": { "op": "feature_set", "colunas": ["rsi", "atr"] }, "entradas": ["ct://derived/btc_ml_feats"] },
      { "id": "lags",       "componente": { "op": "gerar_lags", "colunas": ["rsi", "atr"], "lags": [1, 2, 3] }, "entradas": ["$feats"] },
      { "id": "feats_lag",  "componente": { "op": "aplicar_em_features" }, "entradas": ["$lags", "$feats"] },
      { "id": "alvo",       "componente": { "op": "target_direcao", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "ds",         "componente": { "op": "dataset" }, "entradas": ["$feats_lag", "$alvo"] },
      { "id": "wf",         "componente": { "op": "split_walk_forward", "n_folds": 4, "rolling": false, "treino_inicial_frac": 0.5 }, "entradas": ["$ds"] },
      { "id": "modelo",    "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 100, "max_depth": 4, "learning_rate": 0.05 } }, "entradas": ["$ds", "$wf"] },
      { "id": "pred",      "componente": { "op": "prever" }, "entradas": ["$modelo", "$feats_lag"] }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$wf",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

**Resultado:**

| Campo           | Valor                                                        |
|-----------------|--------------------------------------------------------------|
| modelo_uri      | `ct://models/btc_gbdt_dir5`                                  |
| predicao_uri    | `ct://derived/btc_gbdt_dir5_pred`                            |
| familia         | `gbdt`                                                       |
| colunas_x       | `rsi, atr, rsi_lag1, rsi_lag2, rsi_lag3, atr_lag1, atr_lag2, atr_lag3` |
| classes         | `[-1, 0, 1]`                                                 |
| nos_executados  | 8                                                            |

---

## O que cada nó faz

| Nó          | op                | Função                                                              |
|-------------|-------------------|---------------------------------------------------------------------|
| `feats`     | `feature_set`     | Seleciona colunas (`rsi`, `atr`) do indicador materializado         |
| `lags`      | `gerar_lags`      | Cria defasagens de 1, 2 e 3 barras para cada feature                |
| `feats_lag` | `aplicar_em_features` | Combina lags com features originais num conjunto unificado     |
| `alvo`      | `target_direcao`  | Gera alvo de direção (horizonte=5, limiar=0.0): +1, 0 ou -1         |
| `ds`        | `dataset`         | Monta o dataset (X = features + lags, y = alvo)                     |
| `wf`        | `split_walk_forward` | Divide em 4 folds walk-forward, treino inicial 50%             |
| `modelo`    | `modelo`          | Treina GBDT (100 árvores, profundidade 4, LR 0.05)                  |
| `pred`      | `prever`          | Gera predições usando o modelo treinado sobre as features          |

---

## Passo 4 — Backtest da predição

A URI de predição contém uma coluna `pred` (-1, 0 ou 1). Usamos `ct_backtest` com a predição como indicador:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_gbdt_dir5_pred",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "gbdt_bt_fee"
  }
}
```

---

## Resultado real

| Métrica           | Com fee (0.1%)      | Sem fee             |
|-------------------|---------------------|---------------------|
| Trades            | 435                 | 435                 |
| PnL líquido       | **-$29 999.25**     | **+$25 971.95**     |
| PnL bruto         | +$25 932.87         | +$25 932.87         |
| Total de fees     | $55 843.66          | —                   |
| Sharpe            | 0.024               | 0.142               |
| Win rate          | 20.7%               | 69.4%               |
| Profit factor     | 0.337               | 3.357               |
| Max drawdown      | 30.0%               | —                   |
| Exposição         | 99.7%               | 99.7%               |
| Avg win           | +$168.71            | +$168.71            |
| Avg loss          | -$130.71            | -$130.71            |
| Payoff            | 1.291               | 1.291               |
| Long / Short      | 218 / 217           | 218 / 217           |

---

## O dilema do ML com taxas

O resultado revela um problema clássico de ML aplicado a trading:

- **Edge bruto massivo:** +$25.9K de PnL bruto, win rate 69.4% sem fee, PF = 3.357 — o modelo é extraordinariamente preditivo.
- **Armadilha de taxas:** 435 trades geram $55.8K em fees — **2× maior que o edge**. O PnL líquido afunda para -$30K.
- **Causa raiz:** exposição de 99.7% significa que o modelo opera em praticamente toda barra. Ele prediz direção corretamente mas não filtra ruído — "churna" capital.
- **Problema de fundo:** o modelo usa classes discretas (-1, 0, 1) sem threshold de confiança. Toda predição vira trade, mesmo quando o modelo está incerto.

### Soluções sugeridas

1. **Threshold de predição:** só operar se `|pred|` ou a probabilidade exceder um nível mínimo (ex.: `prob > 0.6`).
2. **Usar probabilidades:** em vez da classe dura, acessar `predict_proba` e negociar apenas acima de uma confiança mínima.
3. **Aumentar o horizonte (`horizonte`):** horizontes maiores reduzem a frequência de trades naturalmente.
4. **Filtro adicional:** usar ATR ou volatilidade para evitar trading em condições de mercado lateral.

---

## Variações

- **Adicionar features:** incluir `ema`, `bollinger` ou `stochastic` na receita do `materializar_indicador` para enriquecer o sinal.
- **Aumentar horizonte:** testar `horizonte: 10` ou `horizonte: 15` para reduzir frequência de trades e optionally capturar movimentos maiores.
- **Adicionar scaler:** inserir um nó `scaler_zscore` antes do `dataset` para normalizar features (útil para modelos lineares; GBDT é invariante a escala mas não prejudica).
- **Usar probabilidades:** substituir `prever` por um nó que extraia `predict_proba` e aplicar threshold de confiança no script de backtest.
