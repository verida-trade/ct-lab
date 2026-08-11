# Ops Declarativas da Pipeline

> Referência detalhada de cada operação declarativa disponível no `montar_pipeline_indicadores`.

---

## `combinar_aritmetica` — `+ − × ÷`

Combina N parcelas (séries ou escalares) em uma única coluna de saída.

```json
{
  "id": "soma_rsi_mfi",
  "operacao": "combinar_aritmetica",
  "operador": "somar",
  "parcelas": [
    { "fonte": "$rsi" },
    { "fonte": "$mfi" }
  ],
  "coluna_saida": "rsi_plus_mfi"
}
```

| Operador | Símbolo |
|---|---|
| `somar` | `+` |
| `subtrair` | `−` |
| `multiplicar` | `×` |
| `dividir` | `÷` |

Cada parcela pode ser:
- `{ "fonte": "$<id>", "coluna": "nome" }` — série de um step anterior
- `{ "escalar": 1.5 }` — número literal

---

## `comparar` — série×série → 0/1

Compara duas séries e produz 1.0 ou 0.0.

```json
{
  "id": "cruz",
  "operacao": "comparar",
  "esquerda": "$sma_curta",
  "direita": "$sma_longa",
  "operador": "cruza_acima"
}
```

| Operador | Significado |
|---|---|
| `maior` | esquerda > direita (barra a barra) |
| `menor` | esquerda < direita |
| `maior_igual` | esquerda ≥ direita |
| `menor_igual` | esquerda ≤ direita |
| `cruza_acima` | esquerda cruza direita de baixo pra cima |
| `cruza_abaixo` | esquerda cruza direita de cima pra baixo |

> **Atenção:** `comparar` é série×série. Para comparar série vs escalar (`rsi > 30`), use `condicional`.

---

## `condicional` — ternário

```json
{
  "id": "filtro",
  "operacao": "condicional",
  "condicao": "$rsi",
  "coluna_condicao": "rsi",
  "entao": { "escalar": 1.0 },
  "senao": { "escalar": 0.0 },
  "coluna_saida": "comprado"
}
```

- `condicao` é uma série (referência `$<id>`). Valor ≠ 0 = verdadeiro.
- `coluna_condicao` — qual coluna da série usar (default: `"close"`)
- `entao` / `senao` — podem ser série (`{ "fonte": "$x" }`) ou escalar (`{ "escalar": 1.0 }`)

---

## `transformar` — função unária

```json
{
  "id": "log_vol",
  "operacao": "transformar",
  "source": "$anchor",
  "column": "volume",
  "funcao": "log",
  "coluna_saida": "log_volume"
}
```

| Função | O que faz |
|---|---|
| `abs` | Valor absoluto |
| `neg` | Negação (−x) |
| `log` | Logaritmo natural |
| `sqrt` | Raiz quadrada |
| `sinal` | Sinal: −1, 0, ou +1 |
| `clamp` | Limita entre `minimo` e `maximo` |

Para `clamp`, passe `minimo` e `maximo`:
```json
{ "funcao": "clamp", "minimo": 0.0, "maximo": 100.0 }
```

---

## `estatistica_rolling` — janela móvel

```json
{
  "id": "desvio_rsi",
  "operacao": "estatistica_rolling",
  "source": "$rsi",
  "metodo": "desvio_padrao",
  "periodo": 20
}
```

| Método | O que calcula |
|---|---|
| `rma` | Wilder's Running Moving Average |
| `smm` | Simple Moving Median |
| `desvio_padrao` | Standard deviation rolante |
| `regressao_linear` | Regressão linear rolante (slope) |

---

## `compose` — join cross-step

```json
{
  "id": "merge",
  "operacao": "compose",
  "columns": [
    { "source": "$btc", "source_column": "close", "as_column": "btc_close" },
    { "source": "$eth", "source_column": "close", "as_column": "eth_close" }
  ]
}
```

Alinha múltiplos steps por timestamp. Útil para cross-asset dentro da pipeline.

---

## `custom` — script Rhai ou Python

```json
{
  "id": "meu_sinal",
  "operacao": "custom",
  "script": "let r = rsi(close, 14); if r[0] > 70.0 { -1.0 } else if r[0] < 30.0 { 1.0 } else { 0.0 }",
  "entradas": [
    { "alias": "close", "fonte": "$anchor", "coluna": "close" }
  ],
  "parametros": {},
  "coluna_saida": "sinal_custom"
}
```

Para Python, use `uri` ou `script` com sintaxe Python. O ambiente roda via `uv`.

---

> Próximo: [Rhai vetorizado](./05-rhai-vetorizado.pt.md) · [Cookbook](./07-cookbook.pt.md)
