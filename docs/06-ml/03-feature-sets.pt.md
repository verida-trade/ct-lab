# Feature Sets

> Puxe colunas nomeadas de qualquer série (raw ou derivada) para formar a matriz de features.

O `feature_set` é o primeiro nó do DAG de ML. Ele lê colunas de uma série (URI) e as transforma em features.

---

## Uso

```json
{
  "id": "features",
  "componente": {
    "colunas": ["rsi", "sma", "macd_hist"]
  },
  "entradas": ["ct://derived/meus_indicadores"]
}
```

- `colunas` (opcional): lista de colunas a usar como features. Default: todas, em ordem.
- `entradas`: URI da série (raw ou derived) de onde puxar as colunas.

---

## A ponte com backtest

A MESMA série derivada (output da Composição) é consumida de forma idêntica por:

- **Backtest:** `indicadores=<uri>` → lê colunas por alias: `ind["rsi"][0]`
- **ML:** `feature_set colunas=["rsi"]` → lê as mesmas colunas como features

```
ct://derived/meus_indicadores[rsi, sma, macd_hist]
     ↓                              ↓
  backtest                        ML (feature_set)
  ind["rsi"][0]                  features["rsi"]
```

Esta é a "ponte" central da plataforma: o artefato é o mesmo, o consumo é por lentes diferentes.

---

> Próximo: [Targets](./04-targets.pt.md)
