# Receita 06 — Treinar Modelo de Direção (GBDT)

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [ML](../docs/06-ml/)

Pipeline completo: features → target direção → split walk-forward → GBDT → avaliar.

## Passo 1 — Buscar série e criar indicadores

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

Criar features com Rhai vetorizado:

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_features",
    "receita": "#{\"rsi\": rsi(close, 14), \"atr\": atr(high, low, close, 14), \"momentum\": close - close[5]}"
  }
}
```

## Passo 2 — Montar esteira ML

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_dir_5",
    "nos": [
      { "id": "features", "componente": { "colunas": ["rsi", "atr", "momentum"] }, "entradas": ["ct://derived/btc_features"] },
      { "id": "lags", "componente": { "colunas": ["rsi", "atr", "momentum"], "lags": [1, 2, 3] }, "entradas": ["$features"] },
      { "id": "target", "componente": { "coluna": "close", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": { "manter_features_nan": false }, "entradas": ["$lags", "$target"] },
      { "id": "split", "componente": { "n_folds": 4, "treino_inicial_frac": 0.5, "rolling": false }, "entradas": ["$dataset"] },
      { "id": "scaler", "componente": {}, "entradas": ["$split"] },
      { "id": "modelo", "componente": { "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 100, "max_depth": 4, "learning_rate": 0.05 } }, "entradas": ["$scaler"] },
      { "id": "avaliar", "componente": {}, "entradas": ["$modelo"] }
    ],
    "modelo": "$modelo",
    "predicao": "$modelo",
    "avaliacao": "$avaliar"
  }
}
```

## Passo 3 — Serving

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "ct://models/btc_dir_5",
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "probas": true
  }
}
```

## Passo 4 — Ponte com backtest

Adicione `avalar_backtest` na esteira:

```json
{
  "avalar_backtest": {
    "capital_inicial": 1000,
    "fee_pct": 0.001
  }
}
```

Estratégia default: long se pred > 0, short se pred < 0.
