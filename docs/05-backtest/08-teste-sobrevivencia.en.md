# Survival Test (Grid)

> **Premium.** The pair test: fires buy AND sell at the same N moments with the same manager. If the layer doesn't bleed even with coin flips, any setup with edge becomes profit on top of the floor.

The foundation of the CT method: **arbitrary-side survival**. Execution cannot depend on getting the direction right.

---

## What it is

The test "pair" = **two independent executions** (buy and sell) over the same N moments. The directional term cancels by construction — what's left measures **execution**, not the guess.

---

## Call

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 20,
    "fee_pct": 0,
    "stop_r": 0.5,
    "ativacao_r": 1.0,
    "dist_r": 0.3,
    "prazo": 128,
    "breakeven": true,
    "reescala_vol": true,
    "piramide": false
  }
}
```

### Parameters

| Parameter | Default | Description |
|---|---|---|
| `serie` | required | OHLCV series URI |
| `momentos` | 20 | N moments spaced in the period |
| `fee_pct` | 0 | Fee per trade (0 = no fees) |
| `stop_r` | 0.5 | Stop in régua units (local amplitude) |
| `ativacao_r` | 1.0 | Trailing activation in réguas |
| `dist_r` | 0.3 | Trailing distance in réguas |
| `prazo` | 128 | Close at market after N bars |
| `breakeven` | false | Move stop to entry when favorable |
| `reescala_vol` | false | Rescale with local volatility |
| `piramide` | false | Pyramid out of high vol |

---

## Return

```json
{
  "soma_pnl": 0.034,
  "ev_par_reguas": 0.063,
  "pares_positivos": 12,
  "por_momento": [...]
}
```

| Field | Meaning |
|---|---|
| `soma_pnl` | Σ net (no fees) — the verdict |
| `ev_par_reguas` | EV per pair in régua units |
| `pares_positivos` | How many of N moments had sum ≥ 0 |
| `por_momento` | P&L of each execution (buy and sell) per moment |

---

## Verdict

- **Σ ≥ 0 (no fees):** the floor exists. The execution layer doesn't bleed.
- **Σ < 0:** the problem is execution/management or the asset has no structure.
- **With fees:** if Σ < 0 with fee, the fat edge must come from the setup.

---

## In the CT Lab UI

The **Grid** tab in the backtest screen does the same test with two clicks: choose the period by date, toggle the adaptive rule flags, see the verdict and per-moment table.

---

> Next: [Comparing backtests](./09-comparacao.en.md) · [CT Doctrine](../08-doutrina/)
