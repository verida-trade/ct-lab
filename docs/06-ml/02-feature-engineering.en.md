# Feature Engineering

> Generate features automatically from series — lags, calendar, transforms and interactions.

Feature generators operate in the `FeatureSet` branch of the DAG, **before** the `dataset`. Each emits an `Ajuste::Inline` (config, not fitted state) that is reapplied at serving time.

---

## `gerar_lags`

Deferrals `x[t-k]` of existing columns:

```json
{
  "componente": { "colunas": ["rsi", "sma"], "lags": [1, 2, 3, 5, 10] }
}
```

Generates: `rsi_lag1`, `rsi_lag2`, ..., `sma_lag1`, `sma_lag2`, ...

---

## `features_calendario`

Cyclic temporal features from timestamp:

```json
{
  "componente": { "campos": ["hora_dia", "dia_semana", "hora_sin", "hora_cos"] }
}
```

| Field | Generates |
|---|---|
| `hora_dia` | Hour of day (0-23) |
| `dia_semana` | Day of week (0-6) |
| `hora_sin`/`hora_cos` | Sine/cosine of hour (cyclic) |
| `dia_sin`/`dia_cos` | Sine/cosine of day (cyclic) |

---

## `transformar_coluna`

Stateless transforms on columns:

```json
{
  "componente": {
    "transformacoes": [
      { "coluna": "volume", "op": "log" },
      { "coluna": "close", "op": "retorno_pct" }
    ]
  }
}
```

| Operation | What it does |
|---|---|
| `log` | Natural logarithm |
| `sqrt` | Square root |
| `sinal` | Sign: -1, 0, +1 |
| `retorno_pct` | Percent return: `(x[t] - x[t-1]) / x[t-1]` |
| `abs` | Absolute value |

---

## `interagir_colunas`

Combine column pairs:

```json
{
  "componente": {
    "pares": [
      { "a": "rsi", "b": "adx", "op": "razao" },
      { "a": "volume", "b": "close", "op": "produto" }
    ]
  }
}
```

| Operation | Result |
|---|---|
| `razao` | `a / b` |
| `produto` | `a * b` |
| `diferenca` | `a - b` |

---

## DAG order

```
[feature_set] → [gerar_lags] → [features_calendario] → [transformar_coluna] → [interagir_colunas] → [dataset]
```

Generators are **causal** — only use data up to current bar. Invalid domain (log of negative, div by zero) → NaN, dropped by `dataset` or treated by `imputar`/`preencher_temporal` downstream.

---

> Next: [Feature sets](./03-feature-sets.en.md)
