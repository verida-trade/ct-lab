# Preprocessing

> Limpeza, normalização, seleção e redução de dimensionalidade.

---

## Limpeza

### `imputar`

Substitui NaN/inf por um valor estimado:

```json
{ "componente": { "estrategia": "media" } }
```

| Estratégia | Descrição |
|---|---|
| `media` | Média das colunas (default) |
| `mediana` | Mediana |
| `zero` | Substitui por 0 |
| `constante` | Substitui por `valor` |

### `preencher_temporal`

Preenche NaN propagando valores:

```json
{ "componente": { "metodo": "ffill" } }
```

| Método | Descrição |
|---|---|
| `ffill` | Forward fill (passado→presente, causal) |
| `bfill` | Backward fill (futuro→presente, NÃO causal) |

### `winsorize`

Limita outliers por quantis:

```json
{ "componente": { "q_inf": 0.01, "q_sup": 0.99 } }
```

---

## Normalização (Scalers)

### `scaler_zscore`

```json
{ "componente": { "metodo": "zscore" } }
```

Normaliza para média=0, desvio=1.

### Outros scalers

| Scaler | O que faz |
|---|---|
| `minmax` | Coloca em [0, 1] |
| `robust` | Mediana + IQR (robusto a outliers) |
| `maxabs` | Divide pelo valor absoluto máximo |

> Scalers são **fit-dependentes**: o ajuste (média/desvio) é calculado no treino e reaplicado no serving.

---

## Seleção

### `selecionar_correlacao`

Mantém features com maior `|corr|` com o target:

```json
{ "componente": { "top_k": 10 } }
```

### `selecionar_variancia`

Mantém features com variância > limiar:

```json
{ "componente": { "limiar": 0.0 } }
```

---

## Redução

### `reduzir_pca`

Reduz dimensionalidade por PCA:

```json
{ "componente": { "n_componentes": 10 } }
```

---

> Próximo: [Modelos built-in](./07-modelos-built-in.pt.md)
