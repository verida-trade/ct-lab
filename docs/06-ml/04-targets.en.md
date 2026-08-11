# Targets

> Define what the model will predict: direction, return, joint classes, or custom Python target.

---

## `target_direcao`

Predicts price direction N bars ahead (classification: 0=down, 1=up):

```json
{
  "componente": { "coluna": "close", "horizonte": 5, "limiar": 0.0 }
}
```

| Parameter | Default | Description |
|---|---|---|
| `coluna` | `"close"` | Price column |
| `horizonte` | 1 | Bars ahead |
| `limiar` | 0.0 | Dead band: `|ret| <= limiar` → class 0 |

---

## `target_retorno`

Predicts continuous return N bars ahead (regression):

```json
{
  "componente": { "coluna": "close", "horizonte": 5, "log": false }
}
```

| Parameter | Default | Description |
|---|---|---|
| `coluna` | `"close"` | Price column |
| `horizonte` | 1 | Bars ahead |
| `log` | false | Uses log-return `ln(p1/p0)` instead of `(p1-p0)/p0` |

---

## `target_custom` (Python)

User-defined target via Python script:

```json
{
  "componente": {
    "script": "def rotular(colunas, hp):\n    ...",
    "deps": ["numpy>=2.0"]
  }
}
```

---

## `target_conjunto`

Combines multiple categorical columns at T+N (e.g., direction × volatility → 9 classes):

```json
{
  "componente": {
    "colunas": ["direcao_3class", "vol_regime_3class"],
    "cardinalidades": [3, 3],
    "horizonte": 1
  }
}
```

Generates `3 × 3 = 9` combined classes.

---

> Next: [Splits](./05-splits.en.md)
