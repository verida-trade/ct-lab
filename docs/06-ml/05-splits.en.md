# Splits

> How to partition the dataset into train/validation/test without leakage.

---

## `split_holdout`

Reserves a final fraction for validation:

```json
{ "componente": { "treino_frac": 0.7 } }
```

Last 30% of lines (in temporal order) become validation.

---

## `split_walk_forward`

Walk-forward validation with successive windows:

```json
{ "componente": { "n_folds": 4, "treino_inicial_frac": 0.5, "rolling": false } }
```

| Parameter | Default | Description |
|---|---|---|
| `n_folds` | 4 | Number of validation windows |
| `treino_inicial_frac` | 0.5 | Initial fraction for 1st train |
| `rolling` | false | `false` = expanding; `true` = rolling (fixed window) |

---

## `split_purged_kfold`

K-fold with purging (embargo) between train and validation:

```json
{ "componente": { "k": 5, "embargo": 10 } }
```

| Parameter | Default | Description |
|---|---|---|
| `k` | 5 | Number of folds |
| `embargo` | 0 | Lines purged from each side of validation block |

Embargo removes observations adjacent to the validation fold to prevent temporal leakage.

---

## `split_custom` (Python)

User-defined split:

```json
{
  "componente": {
    "script": "def partitionar(n_linhas, timestamps, hp):\n    ..."
  }
}
```

---

> Next: [Preprocessing](./06-preprocessing.en.md)
