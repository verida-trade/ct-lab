# Preprocessing

> Cleaning, normalization, selection and dimensionality reduction.

---

## Cleaning

### `imputar`

Replaces NaN/inf with an estimated value:

```json
{ "componente": { "estrategia": "media" } }
```

| Strategy | Description |
|---|---|
| `media` | Mean of columns (default) |
| `mediana` | Median |
| `zero` | Replace with 0 |
| `constante` | Replace with `valor` |

### `preencher_temporal`

Fills NaN by propagating values:

```json
{ "componente": { "metodo": "ffill" } }
```

| Method | Description |
|---|---|
| `ffill` | Forward fill (past→present, causal) |
| `bfill` | Backward fill (future→present, NOT causal) |

### `winsorize`

Limits outliers by quantiles:

```json
{ "componente": { "q_inf": 0.01, "q_sup": 0.99 } }
```

---

## Normalization (Scalers)

### `scaler_zscore`

```json
{ "componente": { "metodo": "zscore" } }
```

Normalizes to mean=0, std=1.

### Other scalers

| Scaler | What it does |
|---|---|
| `minmax` | Scales to [0, 1] |
| `robust` | Median + IQR (outlier-robust) |
| `maxabs` | Divide by max absolute value |

> Scalers are **fit-dependent**: the adjustment (mean/std) is computed on train and reapplied at serving.

---

## Selection

### `selecionar_correlacao`

Keep features with highest `|corr|` with target:

```json
{ "componente": { "top_k": 10 } }
```

### `selecionar_variancia`

Keep features with variance > threshold:

```json
{ "componente": { "limiar": 0.0 } }
```

---

## Reduction

### `reduzir_pca`

Reduce dimensionality by PCA:

```json
{ "componente": { "n_componentes": 10 } }
```

---

> Next: [Built-in models](./07-modelos-built-in.en.md)
