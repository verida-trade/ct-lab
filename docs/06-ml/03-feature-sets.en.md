# Feature Sets

> Pull named columns from any series (raw or derived) to form the feature matrix.

`feature_set` is the first node in the ML DAG. It reads columns from a series (URI) and turns them into features.

---

## Usage

```json
{
  "id": "features",
  "componente": { "colunas": ["rsi", "sma", "macd_signal"] },
  "entradas": ["ct://derived/my_indicators"]
}
```

- `colunas` (optional): list of columns to use as features. Default: all, in order.
- `entradas`: URI of the series (raw or derived) to pull columns from.

---

## The bridge with backtest

The SAME derived series (Composition output) is consumed identically by:

- **Backtest:** `indicadores=<uri>` → reads columns by alias: `ind["rsi"][0]`
- **ML:** `feature_set colunas=["rsi"]` → reads same columns as features

```
ct://derived/my_indicators[rsi, sma, macd_signal]
     ↓                              ↓
  backtest                        ML (feature_set)
  ind["rsi"][0]                  features["rsi"]
```

This is the central "bridge" of the platform: the artifact is the same, consumption is by different lenses.

---

> Next: [Targets](./04-targets.en.md)
