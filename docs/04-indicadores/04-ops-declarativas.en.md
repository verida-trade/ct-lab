# Declarative Pipeline Ops

> Detailed reference for each declarative operation available in `montar_pipeline_indicadores`.

---

## `combinar_aritmetica` — `+ − × ÷`

Combines N operands (series or scalars) into a single output column.

```json
{
  "id": "add_rsi_mfi",
  "op": "combinar_aritmetica",
  "operador": "somar",
  "parcelas": [
    { "fonte": "$rsi" },
    { "fonte": "$mfi" }
  ],
  "coluna_saida": "rsi_plus_mfi"
}
```

| Operator | Symbol |
|---|---|
| `somar` | `+` |
| `subtrair` | `−` |
| `multiplicar` | `×` |
| `dividir` | `÷` |

Each operand can be:
- `{ "fonte": "$<id>", "coluna": "name" }` — series from a prior step
- `{ "escalar": 1.5 }` — literal number

---

## `comparar` — series×series → 0/1

Compares two series and produces 1.0 or 0.0.

```json
{
  "id": "cross",
  "op": "comparar",
  "esquerda": "$sma_short",
  "direita": "$sma_long",
  "operador": "cruza_acima"
}
```

| Operator | Meaning |
|---|---|
| `maior` | left > right (bar by bar) |
| `menor` | left < right |
| `maior_igual` | left ≥ right |
| `menor_igual` | left ≤ right |
| `cruza_acima` | left crosses right from below |
| `cruza_abaixo` | left crosses right from above |

> **Warning:** `comparar` is series×series. To compare series vs scalar (`rsi > 30`), use `condicional`.

---

## `condicional` — ternary

```json
{
  "id": "filter",
  "op": "condicional",
  "condicao": "$rsi",
  "coluna_condicao": "rsi",
  "entao": { "escalar": 1.0 },
  "senao": { "escalar": 0.0 },
  "coluna_saida": "long"
}
```

- `condicao` is a series (`$<id>` reference). Value ≠ 0 = true.
- `coluna_condicao` — which column to use (default: `"close"`)
- `entao` / `senao` — can be series (`{ "fonte": "$x" }`) or scalar (`{ "escalar": 1.0 }`)

---

## `transformar` — unary function

```json
{
  "id": "log_vol",
  "op": "transformar",
  "source": "$anchor",
  "column": "volume",
  "funcao": "log",
  "coluna_saida": "log_volume"
}
```

| Function | What it does |
|---|---|
| `abs` | Absolute value |
| `neg` | Negation (−x) |
| `log` | Natural logarithm |
| `sqrt` | Square root |
| `sinal` | Sign: −1, 0, or +1 |
| `clamp` | Clamp between `minimo` and `maximo` |

For `clamp`, pass `minimo` and `maximo`:
```json
{ "funcao": "clamp", "minimo": 0.0, "maximo": 100.0 }
```

---

## `estatistica_rolling` — rolling window

```json
{
  "id": "rsi_std",
  "op": "estatistica_rolling",
  "source": "$rsi",
  "metodo": "desvio_padrao",
  "periodo": 20
}
```

| Method | What it computes |
|---|---|
| `rma` | Wilder's Running Moving Average |
| `smm` | Simple Moving Median |
| `desvio_padrao` | Rolling standard deviation |
| `regressao_linear` | Rolling linear regression (slope) |

---

## `compose` — cross-step join

```json
{
  "id": "merge",
  "op": "compose",
  "columns": [
    { "source": "$btc", "source_column": "close", "as_column": "btc_close" },
    { "source": "$eth", "source_column": "close", "as_column": "eth_close" }
  ]
}
```

Aligns multiple steps by timestamp. Useful for cross-asset within the pipeline.

---

## `custom` — Rhai or Python script

```json
{
  "id": "my_signal",
  "op": "custom",
  "script": "let r = rsi(close, 14); if r[0] > 70.0 { -1.0 } else if r[0] < 30.0 { 1.0 } else { 0.0 }",
  "entradas": [
    { "alias": "close", "fonte": "$anchor", "coluna": "close" }
  ],
  "parametros": {},
  "coluna_saida": "custom_signal"
}
```

For Python, use `uri` or `script` with Python syntax. The environment runs via `uv`.

---

> Next: [Vectorized Rhai](./05-rhai-vetorizado.en.md) · [Cookbook](./07-cookbook.en.md)
