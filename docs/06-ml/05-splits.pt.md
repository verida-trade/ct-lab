# Splits

> Como particionar o dataset em treino/validação/teste sem leakage.

---

## `split_holdout`

Reserva uma fração final para validação:

```json
{ "componente": { "treino_frac": 0.7 } }
```

Últimos 30% das linhas (em ordem temporal) viram validação.

---

## `split_walk_forward`

Validação walk-forward com janelas sucessivas:

```json
{
  "componente": {
    "n_folds": 4,
    "treino_inicial_frac": 0.5,
    "rolling": false
  }
}
```

| Parâmetro | Default | Descrição |
|---|---|---|
| `n_folds` | 4 | Número de janelas de validação |
| `treino_inicial_frac` | 0.5 | Fração inicial para 1º treino |
| `rolling` | false | `false` = expanding (treino cresce); `true` = rolling (janela fixa) |

---

## `split_purged_kfold`

K-fold com purging (embargo) entre treino e validação:

```json
{
  "componente": {
    "k": 5,
    "embargo": 10
  }
}
```

| Parâmetro | Default | Descrição |
|---|---|---|
| `k` | 5 | Número de folds |
| `embargo` | 0 | Linhas purgadas de cada lado do bloco de validação |

O embargo remove observações adjacentes ao fold de validação para evitar leakage temporal.

---

## `split_custom` (Python)

Split definido pelo usuário:

```json
{
  "componente": {
    "script": "def partitionar(n_linhas, timestamps, hp):\n    folds = []\n    fold_size = n_linhas // 5\n    for i in range(5):\n        val_start = i * fold_size\n        val_end = (i + 1) * fold_size\n        train = list(range(0, val_start)) + list(range(val_end, n_linhas))\n        val = list(range(val_start, val_end))\n        folds.append({'treino': train, 'validacao': val})\n    return folds"
  }
}
```

---

> Próximo: [Preprocessing](./06-preprocessing.pt.md)
