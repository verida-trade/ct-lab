# Feature Engineering

> Gere features automaticamente a partir de séries — lags, calendário, transformações e interações.

Os geradores de features operam no ramo `FeatureSet` do DAG, **antes** do `dataset`. Cada um emite um `Ajuste::Inline` (config, não estado fitado) que é reaplicado no serving.

---

## `gerar_lags`

Defasagens `x[t-k]` de colunas existentes:

```json
{
  "componente": {
    "colunas": ["rsi", "sma"],
    "lags": [1, 2, 3, 5, 10]
  },
  "entradas": ["$features"]
}
```

Gera colunas: `rsi_lag1`, `rsi_lag2`, ..., `sma_lag1`, `sma_lag2`, ...

---

## `features_calendario`

Features temporais cíclicas a partir do timestamp:

```json
{
  "componente": {
    "campos": ["hora_dia", "dia_semana", "hora_sin", "hora_cos", "dia_sin", "dia_cos"]
  }
}
```

| Campo | O que gera |
|---|---|
| `hora_dia` | Hora do dia (0-23) |
| `dia_semana` | Dia da semana (0-6) |
| `hora_sin`/`hora_cos` | Seno/cosseno da hora (cíclico) |
| `dia_sin`/`dia_cos` | Seno/cosseno do dia (cíclico) |

---

## `transformar_coluna`

Transformações sem fit (stateless) sobre colunas:

```json
{
  "componente": {
    "transformacoes": [
      { "coluna": "volume", "op": "log" },
      { "coluna": "close", "op": "retorno_pct" },
      { "coluna": "rsi", "op": "sinal" }
    ]
  }
}
```

| Operação | O que faz |
|---|---|
| `log` | Logaritmo natural |
| `sqrt` | Raiz quadrada |
| `sinal` | Sinal: -1, 0, +1 |
| `retorno_pct` | Retorno percentual: `(x[t] - x[t-1]) / x[t-1]` |
| `abs` | Valor absoluto |

---

## `interagir_colunas`

Combina pares de colunas:

```json
{
  "componente": {
    "pares": [
      { "a": "rsi", "b": "adx", "op": "razao" },
      { "a": "volume", "b": "close", "op": "produto" },
      { "a": "high", "b": "low", "op": "diferenca" }
    ]
  }
}
```

| Operação | Resultado |
|---|---|
| `razao` | `a / b` |
| `produto` | `a * b` |
| `diferenca` | `a - b` |

---

## Ordem no DAG

```
[feature_set] → [gerar_lags] → [features_calendario] → [transformar_coluna] → [interagir_colunas] → [dataset]
```

Os geradores são **causais** — só usam dados até a barra atual. Domínio inválido (log de negativo, divisão por zero) → NaN, que é dropado pelo `dataset` ou tratado por `imputar`/`preencher_temporal` downstream.

---

> Próximo: [Feature sets](./03-feature-sets.pt.md)
