# Recipe 04 — Backtest with the `grupo` lib

> **Level:** Intermediate · **Plan:** Premium
> **Prerequisite:** [docs/05-backtest/05-lib-grupo](../../../docs/05-backtest/05-lib-grupo)

---

## The `grupo` lib

- **Staggered entries:** defines one or more entry orders with `offset`, `tipo` (limit/market), and `lado` (buy/sell), fired in sequence.
- **OCO exits:** each exit (stop, limit, trailing) competes within the same cycle — the first one to trigger cancels the rest.
- **Trailing stop:** follows price at a fixed `distancia`, activating only when profit reaches `ativacao` points.
- **Cycles:** controls how many times the group re-arms after an exit (`ciclos: 1.0` = re-arm once; `ciclos: 0` = continuous).

---

## Step 1 — Fetch series

```
ct_buscar_binance("BTCUSDT", "15m")
```

Series used: `ct://series/binance/BTCUSDT/15m` (1724 candles).

---

## Step 2 — The strategy

```rhai
import "grupo" as g;

// Group config: 1 limit entry, stop -20, TP +15, trailing 8/10
let cfg = #{
    entradas: [
        #{ offset: 0.0, tipo: "limit", lado: "compra", lote: 1.0 },
    ],
    saidas: [
        #{ offset: -20.0, tipo: "stop", lote: 1.0 },
        #{ offset: 15.0, tipo: "limit", lote: 1.0 },
        #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 1.0 },
    ],
    ciclos: 1.0,  // re-arm once after the first exit
};

// grupo::tick processes the current candle and returns a decision + new state
let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;  // CRITICAL: persist state across bars
r.decisao
```

---

## Step 3 — Backtest

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "import \"grupo\" as g;\nlet cfg = #{entradas:[#{offset:0.0,tipo:\"limit\",lado:\"compra\",lote:1.0}],saidas:[#{offset:-20.0,tipo:\"stop\",lote:1.0},#{offset:15.0,tipo:\"limit\",lote:1.0},#{tipo:\"trailing\",distancia:8.0,ativacao:10.0,lote:1.0}],ciclos:1.0};\nlet r=g::tick(cfg,estado,posicao,open[0],high[0],low[0],close[0],ordens);\nestado=r.estado;\nr.decisao",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "grupo_stop_trailing"
  }
}
```

---

## Real results

| Metric | With fee (0.1%) | No fee |
|---|---|---|
| **Trades** | 1 | 1 |
| **PnL** | −$148.58 | −$20.58 |
| **Gross PnL** | −$20.58 | −$20.58 |
| **Fees** | $128.00 | — |
| **Sharpe** | −0.024 | −0.024 |
| **Win rate** | 0% | 0% |
| **Profit factor** | 0 | 0 |
| **Max drawdown** | 14.86% | 2.06% |
| **Expectancy** | 0.058% | — |
| **Avg loss** | −$148.58 | — |
| **Long** | 1 | 1 |

---

## Why only 1 trade?

- **`ciclos: 1.0`** allows only one re-arm after the first exit. Since the stop was hit in the first cycle, the group re-arms once — but only 1 trade occurs in the period.
- **Very tight stop:** −$20 on BTC at ~$64k is just **0.03%** of price. In a single 15m candle, BTC easily swings $50–$100, so the stop was hit almost immediately.
- **Trailing never activated:** the trailing required price to rise +$10 (ativacao) before it would start tracking. Because the stop fired first, the trailing was never evaluated.
- **Fee impact:** with `lote: 1.0` (1 full BTC) and $1000 capital, the 0.1% per-side fee on ~$64k generates **$128** in fees — 6× larger than the $20 stop loss.

---

## ⚠️ Note

The line `estado = r.estado` is ** mandatory**. Without it, the `estado` variable is never updated between bars, and `grupo` treats every candle as the first one — re-arming orders, losing position, and generating phantom trades. The `estado` carries the current cycle, pending orders, and internal group flags.

---

## Variations

- **Wider stop (−50):** replace `offset: -20.0` with `offset: -50.0` to give the trade more room and avoid premature exit.
- **Continuous execution:** set `ciclos: 0` so the group re-arms indefinitely after each exit.
- **Staggered entry:** add `#{ offset: -10.0, tipo: "limit", lado: "compra", lote: 1.0 }` to the `entradas` array to open position at two levels.
- **Increase lot:** change `lote: 1.0` to `lote: 2.0` and watch how fees and PnL scale proportionally.
