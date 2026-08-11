# Receita 09 — Regime + Modelo Preditivo

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [ML](../docs/06-ml/), [Indicadores premium](../docs/04-indicadores/02-indicadores-premium.pt.md)

Use `ct_tendencia` como feature de regime num modelo de ML.

## Passo 1 — Buscar série e criar ct_tendencia

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{
  "name": "ct_tendencia",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "janela_zscore": 96,
    "janela_atr": 14,
    "k_mov": 3,
    "z_min": 2.0,
    "tau_r": 1.5
  }
}
```

O `ct_tendencia` retorna 5 colunas: `tend_ativa`, `direcao`, `fase`, `progresso`, `nivel_rompido`.

## Passo 2 — Montar esteira ML com regime como feature

```json
{
  "name": "montar_esteira_ml",
  "arguments": {
    "name": "btc_regime_model",
    "nos": [
      { "id": "features", "componente": { "colunas": ["direcao", "fase", "progresso"] }, "entradas": ["ct://derived/ct_tendencia_..."] },
      { "id": "rsi_feat", "componente": { "colunas": ["rsi"] }, "entradas": ["ct://derived/btc_features"] },
      { "id": "lags", "componente": { "colunas": ["direcao", "fase", "progresso", "rsi"], "lags": [1, 2, 3, 5] }, "entradas": ["$features"] },
      { "id": "target", "componente": { "coluna": "close", "horizonte": 5, "limiar": 0.0 }, "entradas": ["ct://series/binance/BTCUSDT/15m"] },
      { "id": "dataset", "componente": {}, "entradas": ["$lags", "$target"] },
      { "id": "split", "componente": { "n_folds": 4, "treino_inicial_frac": 0.5 }, "entradas": ["$dataset"] },
      { "id": "scaler", "componente": {}, "entradas": ["$split"] },
      { "id": "modelo", "componente": { "familia": "gbdt", "tarefa": "classificacao", "hiperparametros": { "n_estimators": 150, "max_depth": 5 } }, "entradas": ["$scaler"] },
      { "id": "avaliar", "componente": {}, "entradas": ["$modelo"] }
    ],
    "modelo": "$modelo",
    "predicao": "$modelo",
    "avaliacao": "$avaliar",
    "avalar_backtest": { "capital_inicial": 1000, "fee_pct": 0.001 }
  }
}
```

## Interpretação

O modelo recebe features de regime (direcao, fase, progresso da tendência) + lags + RSI. A predição é materializada em `ct://derived/btc_regime_model_pred` e pode ser usada como indicador em backtests ou em outras esteiras.

> **Nota:** Regime é SETUP, não fundação. O modelo precisa superar o piso de lado arbitrário no mesmo teste.
