# Receita 10 — Pipeline Completo de ML

> **Nível:** Avançado · **Requer:** Premium · **Pré-requisitos:** Receitas 2 (indicadores), 6 (lags), 7 (dataset), 8 (walk-forward), 9 (modelo)

---

## O pipeline ponta-a-ponta

O `montar_esteira_ml` permite encadear **todos** os passos de um projeto de machine learning num único grafo DAG — da matéria-prima (série + indicadores) até a predição final e avaliação de backtest. Os 11 passos são:

1. **Buscar série** e gerar indicadores (RSI + ATR) com `materializar_indicador`
2. **Feature set** — selecionar colunas de interesse
3. **Gerar lags** — criar variáveis defasadas para janela temporal
4. **Aplicar em features** — mesclar lags com o feature set original
5. **Calendário** — adicionar campos sazonais (hora, dia, senos/cossenos)
6. **Target direção** — definir o alvo binário (alta/baixa) num horizonte
7. **Dataset** — cruzar features + target, alinhar timestamps
8. **Imputar** — preencher NaN (lags geram vazios no início)
9. **Split walk-forward** — dividir treino/validação em `n_folds`
10. **Scaler + Seleção** — normalizar (z-score) e reduzir dimensionalidade
11. **Modelo + Prever** — treinar GBDT e gerar predições pontuais

---

## Passo 1 — Buscar série e features

```python
# 1a. Buscar klines da Binance
buscar_binance(symbol="BTCUSDT", interval="15m", limit=2000)

# 1b. Materializar RSI e ATR sobre a série
materializar_indicador(
    serie="ct://series/binance/BTCUSDT/15m",
    indicador="rsi",
    parametros={"period": 14},
    destino="ct://derived/btc_rsi"
)
materializar_indicador(
    serie="ct://series/binance/BTCUSDT/15m",
    indicador="atr",
    parametros={"period": 14},
    destino="ct://derived/btc_atr"
)

# 1c. Compor série única com as duas features
compor_serie(
    series=["ct://derived/btc_rsi", "ct://derived/btc_atr"],
    destino="ct://derived/btc_ml_feats"
)
```

---

## Passo 2 — Montar esteira completa

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_full_ml",
    "nos": [
      { "id": "feats", "componente": { "op": "feature_set", "colunas": ["rsi", "atr"] }, "entradas": ["ct://derived/btc_ml_feats"] },
      { "id": "lags", "componente": { "op": "gerar_lags", "colunas": ["rsi", "atr"], "lags": [1, 2, 3, 5, 10] }, "entradas": ["$feats"] },
      { "id": "feats_lag", "componente": { "op": "aplicar_em_features" }, "entradas": ["$lags", "$feats"] },
      { "id": "cal", "componente": { "op": "gerar_calendario", "campos": ["hora_dia", "dia_semana", "hora_sin", "hora_cos", "dia_sin", "dia_cos"] }, "entradas": ["$feats_lag"] },
      { "id": "alvo", "componente": { "op": "target_direcao", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "ds", "componente": { "op": "dataset", "manter_features_nan": true }, "entradas": ["$cal", "$alvo"] },
      { "id": "impute", "componente": { "op": "imputar", "estrategia": "media" }, "entradas": ["$ds"] },
      { "id": "wf", "componente": { "op": "split_walk_forward", "n_folds": 5, "rolling": false, "treino_inicial_frac": 0.5 }, "entradas": ["$impute"] },
      { "id": "scaler", "componente": { "op": "scaler_zscore" }, "entradas": ["$wf"] },
      { "id": "select", "componente": { "op": "selecionar_features", "top_k": 15 }, "entradas": ["$scaler"] },
      { "id": "modelo", "componente": { "op": "modelo", "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 200, "max_depth": 5, "learning_rate": 0.05 } }, "entradas": ["$select", "$select"] },
      { "id": "pred", "componente": { "op": "prever" }, "entradas": ["$modelo", "$feats_lag"] }
    ],
    "modelo": "$modelo",
    "predicao": "$pred",
    "avaliacao": "$select",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

> **Nota sobre o nó `modelo`:** recebe `["$select", "$select"]` — o mesmo nó de seleção de features fornece os splits de treino e validação (o DAG resolve o walk-forward internamente).

---

## O que cada nó faz

| Nó | `op` | Função |
|---|---|---|
| `feats` | `feature_set` | Seleciona colunas (`rsi`, `atr`) da série composta |
| `lags` | `gerar_lags` | Cria defasagens [1, 2, 3, 5, 10] para cada coluna |
| `feats_lag` | `aplicar_em_features` | Mescla lags gerados com as features originais |
| `cal` | `gerar_calendario` | Adiciona 6 campos sazonais (hora, dia, sin/cos) |
| `alvo` | `target_direcao` | Define alvo binário: subiu (1) ou desceu (0) em 5 barras |
| `ds` | `dataset` | Cruza features + alvo; `manter_features_nan` preserva NaN dos lags |
| `impute` | `imputar` | Preenche NaN com a média (`estrategia": "media"`) |
| `wf` | `split_walk_forward` | Divide em 5 folds walk-forward, treino inicial 50% |
| `scaler` | `scaler_zscore` | Normaliza features para média 0, desvio 1 |
| `select` | `selecionar_features` | Mantém as top 15 features mais relevantes |
| `modelo` | `modelo` | Treina GBDT (`n_estimators=200`, `max_depth=5`) |
| `pred` | `prever` | Gera predições pontuais a partir do modelo + features |

---

## Passo 3 — Backtest da predição

A predição (`$pred`) é automaticamente transformada num pseudo-indicador pela esteira. Para backtestar manualmente:

```python
ct_backtest(
    serie="ct://series/binance/BTCUSDT/15m",
    indicador="ct://predictions/btc_full_ml",
    capital_inicial=1000,
    fee_pct=0.001
)
```

O campo `avalar_backtest` na configuração da esteira já dispara essa avaliação automaticamente ao final do `montar_esteira_ml`.

---

## Por que cada pré-processamento importa

- **Imputar** — Lags de janela 10 geram 10 linhas de NaN no início. Sem imputação, essas linhas são descartadas e o dataset perde dados preciosos de treino.
- **Scaler z-score** — Modelos lineares (regressão logística) e redes neurais (MLP) exigem normalização. GBDT não precisa, mas não prejudica e mantém o pipeline genérico.
- **Seleção de features** — Com lags de 2 colunas × 5 janelas + 6 campos de calendário = 16+ features. Reduzir a top 15 diminui overfitting e acelera treino.
- **Calendário** — Mercados cripto têm padrões intradiários (ex.: volume maior em horários específicos). Senos/cossenos codificam cyclicalidade sem explosão de dimensionalidade.
- **Walk-forward** — Validação temporal evita data leakage. `rolling=false` com `treino_inicial_frac=0.5` expande o treino a cada fold.

---

## Variações

- 🔁 **Trocar família do modelo:** substituir `"familia": "gbdt"` por `"familia": "mlp"` e adicionar camadas em `hiperparametros` (`{"hidden_layers": [64, 32], "epochs": 50}`).
- 📐 **Adicionar PCA:** inserir um nó `{ "op": "pca", "n_componentes": 10 }` entre `scaler` e `select` para reduzir dimensionalidade por variância.
- 🔀 **Variar n_folds:** aumentar para `n_folds=10` (mais folds, treino menor por fold) ou diminuir para `3` (menos folds, treino maior, mais robusto por fold).
- 📈 **Regressor em vez de classificador:** trocar `"target_direcao"` por `"target_retorno"` e `"tarefa": "classificacao"` por `"tarefa": "regressao"` para prever magnitude do retorno.
