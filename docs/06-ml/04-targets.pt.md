# Targets

> Defina o que o modelo vai prever: direção, retorno, classes conjuntas ou target custom em Python.

---

## `target_direcao`

Prevê a direção do preço em N barras à frente (classificação: 0=baixa, 1=alta):

```json
{
  "componente": {
    "coluna": "close",
    "horizonte": 5,
    "limiar": 0.0
  },
  "entradas": ["ct://series/binance/BTCUSDT/15m"]
}
```

| Parâmetro | Default | Descrição |
|---|---|---|
| `coluna` | `"close"` | Coluna de preço |
| `horizonte` | 1 | Barras à frente |
| `limiar` | 0.0 | Banda morta: `|ret| <= limiar` → classe 0 |

---

## `target_retorno`

Prevê o retorno contínuo em N barras (regressão):

```json
{
  "componente": {
    "coluna": "close",
    "horizonte": 5,
    "log": false
  }
}
```

| Parâmetro | Default | Descrição |
|---|---|---|
| `coluna` | `"close"` | Coluna de preço |
| `horizonte` | 1 | Barras à frente |
| `log` | false | Usa log-retorno `ln(p1/p0)` em vez de `(p1-p0)/p0` |

---

## `target_custom` (Python)

Target definido pelo usuário via script Python:

```json
{
  "componente": {
    "script": "def rotular(colunas, hp):\n    close = colunas['close']\n    valores = []\n    for i in range(len(close)):\n        if i + hp < len(close):\n            ret = (close[i+hp] - close[i]) / close[i]\n            if ret > 0.02: valores.append(2)  # alta forte\n            elif ret > 0: valores.append(1)   # alta\n            elif ret > -0.02: valores.append(0) # neutro\n            else: valores.append(-1)           # baixa\n        else:\n            valores.append(0)\n    return {'valores': valores, 'horizonte': hp}",
    "deps": ["numpy>=2.0"]
  },
  "entradas": ["ct://series/binance/BTCUSDT/15m"]
}
```

---

## `target_conjunto`

Combina múltiplas colunas categóricas em T+N (ex.: direção × volatilidade → 9 classes):

```json
{
  "componente": {
    "colunas": ["direcao_3class", "vol_regime_3class"],
    "cardinalidades": [3, 3],
    "horizonte": 1
  }
}
```

Gera `3 × 3 = 9` classes combinadas.

---

> Próximo: [Splits](./05-splits.pt.md)
