# Estratégia em Python

> Use Python como linguagem de estratégia — ideal para lógica complexa, ML ou bibliotecas científicas.

O `ct_backtest` aceita estratégias em Python (`.py`) além de Rhai. O ambiente Python roda via `uv` num ambiente efêmero controlado.

---

## Quando usar Python vs Rhai

| Critério | Rhai | Python |
|---|---|---|
| Velocidade | ⚡ In-process | ⏳ Overhead `uv` |
| Bibliotecas | Nenhuma | numpy, pandas, scipy, etc. |
| ML no backtest | Não | Sim |
| Complexidade | Média | Alta |

---

## Estrutura do script

O script Python deve definir uma função `tick` que recebe os dados da barra e retorna a decisão:

```python
import numpy as np

# Estado global (persiste entre barras)
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

## Chamada

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia": "file:///path/to/minha_estrategia.py",
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

## Deps Python

Adicione bibliotecas via `deps` no script (o `uv` resolve):

```python
# /// script
# dependencies = ["numpy>=2.0", "scipy>=1.14"]
# ///

from scipy.signal import find_peaks
import numpy as np
...
```

---

> Próximo: [A lib `grupo`](./05-lib-grupo.pt.md) · [Rhai avançado](./03-estrategia-rhai.pt.md)
