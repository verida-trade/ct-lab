# Python Strategy

> Use Python as the strategy language — ideal for complex logic, ML, or scientific libraries.

`ct_backtest` accepts Python (`.py`) strategies in addition to Rhai. The Python environment runs via `uv` in a controlled ephemeral environment.

---

## When to use Python vs Rhai

| Criterion | Rhai | Python |
|---|---|---|
| Speed | ⚡ In-process | ⏳ `uv` overhead |
| Libraries | None | numpy, pandas, scipy, etc. |
| ML in backtest | No | Yes |
| Complexity | Medium | High |

---

## Script structure

The Python script must define a `tick` function that receives bar data and returns the decision:

```python
import numpy as np

entry_price = 0.0
in_position = False

def tick(close, open_, high, low, volume, posicao, ind, par, ordens):
    global entry_price, in_position

    rsi = ind["rsi"][0]

    if posicao == 0.0 and rsi < par["oversold"]:
        entry_price = close
        in_position = True
        return {"alvo": "comprado", "lote": 1.0}
    elif posicao > 0.0:
        if close < entry_price * (1.0 - par["stop_pct"]):
            return {"alvo": "zerado"}
        return {"alvo": "comprado", "lote": 1.0}
    else:
        return {"alvo": "zerado"}
```

---

## Call

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia": "file:///path/to/my_strategy.py",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "parametros": { "oversold": 30.0, "stop_pct": 0.02 },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "py_rsi_strategy"
  }
}
```

---

## Python deps

Add libraries via `deps` in the script (`uv` resolves them):

```python
# /// script
# dependencies = ["numpy>=2.0", "scipy>=1.14"]
# ///

from scipy.signal import find_peaks
import numpy as np
...
```

---

> Next: [The `grupo` lib](./05-lib-grupo.en.md) · [Advanced Rhai](./03-estrategia-rhai.en.md)
