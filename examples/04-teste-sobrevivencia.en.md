# 04 — Survival Test

> **Level:** Intermediate · **Premium** · **Prerequisites:** [Backtest with the `grupo` Lib](./03-lib-grupo-backtest.en.md), [The `grupo` Lib](../docs/05-backtest/05-lib-grupo.en.md)

The survival test is the fundamental examination of the CT Lab doctrine. It answers an uncomfortable question:

> **If you buy and sell at arbitrary moments — with no entry criterion — how much do you lose?**

The `ct_testar_sobrevivencia` tool fires **N moments** spaced across the period. Each moment opens **buy AND sell** simultaneously, using the same adaptive manager (a fork of the `grupo` lib). The result measures the **arbitrary-side floor**: if the pair (buy + sell) survives without fees, your strategy has surplus to compete. If it doesn't, no entry factor will save it.

In this example, you will:

1. **Run the survival test** with default parameters (20 moments).
2. **Interpret the result** — what each metric means and how to read `por_momento`.
3. **Vary parameters** (stop, pyramid, breakeven, rescale) to see which adaptive rules matter.
4. **Add fees** to see the devastating effect of execution cost.
5. **Conclude** whether your strategy has surplus or not.

---

## The problem

You have a staggered accumulation strategy with stop, take-profit, and trailing. In [example 03](./03-lib-grupo-backtest.en.md) you saw that the backtest of 138 trades had a gross PnL of −$161 (nearly zero) — the survival floor seemed to exist.

But that was **one direction only** (buy). In the real market, you don't know the side. The survival test asks: what if you had bought **and** sold at the same instant, with the same manager? The pair only survives if **the sum of both PnLs is ≥ 0**.

> The régua (ruler) is the adaptive distance unit: 1 régua = average amplitude of the `w_vol` window (64 bars by default). The 0.5-régua stop adjusts to current volatility — in calm periods, the stop is closer; in turbulent periods, wider.

---

## Step 1 — Fetch data

```
Fetch BTCUSDT at 15m from Binance.
```

```json
{
  "name": "buscar_serie",
  "arguments": {
    "provider": "binance",
    "symbol": "BTCUSDT",
    "interval": "15m"
  }
}
```

> The series becomes available at `ct://series/binance/BTCUSDT/15m`.

---

## Step 2 — Run the test (default: 20 moments, no fees)

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m"
  }
}
```

All parameters have doctrine defaults:
- `momentos`: 20 (the classic test)
- `stop_r`: 0.5 régua
- `lib_grupo`: `"grupo"` (the pre-installed lib)
- `piramide`: true, `breakeven`: true, `reescala_vol`: true
- `fee_pct`: 0 (the test is without fees — we measure the floor)
- `w_vol`: 64, `w_tercil`: 480

### Result (1000 candles of BTCUSDT 15m)

```json
{
  "n_momentos": 20,
  "soma_pnl": -3747.39,
  "soma_par_reguas": -1.719,
  "ev_par_reguas": -0.0859,
  "pares_positivos": 8,
  "por_momento": [
    { "ts": 1785447900, "regua": 1233, "pnl_compra":    0, "pnl_venda": -617, "par_reguas": -0.500 },
    { "ts": 1785484800, "regua": 1548, "pnl_compra": -774, "pnl_venda":  280, "par_reguas": -0.319 },
    { "ts": 1785522600, "regua": 2031, "pnl_compra": -329, "pnl_venda":  329, "par_reguas":  0.000 },
    { "ts": 1785559500, "regua": 1383, "pnl_compra": -692, "pnl_venda":    0, "par_reguas": -0.500 },
    { "ts": 1785597300, "regua":  289, "pnl_compra": -145, "pnl_venda":  803, "par_reguas":  2.275 },
    { "ts": 1785635100, "regua":  852, "pnl_compra":    0, "pnl_venda": -636, "par_reguas": -0.747 },
    { "ts": 1785672000, "regua": 1076, "pnl_compra":    0, "pnl_venda": -536, "par_reguas": -0.498 },
    { "ts": 1785709800, "regua":  814, "pnl_compra": -407, "pnl_venda":  660, "par_reguas":  0.311 },
    { "ts": 1785747600, "regua": 1496, "pnl_compra":  975, "pnl_venda": -748, "par_reguas":  0.151 },
    { "ts": 1785784500, "regua": 1760, "pnl_compra":  616, "pnl_venda": -890, "par_reguas": -0.156 },
    { "ts": 1785822300, "regua": 1108, "pnl_compra":  659, "pnl_venda": -537, "par_reguas":  0.110 },
    { "ts": 1785860100, "regua":  922, "pnl_compra":    0, "pnl_venda": -461, "par_reguas": -0.500 },
    { "ts": 1785897000, "regua":  934, "pnl_compra": -467, "pnl_venda":    0, "par_reguas": -0.500 },
    { "ts": 1785934800, "regua":  631, "pnl_compra":  451, "pnl_venda": -316, "par_reguas":  0.215 },
    { "ts": 1785972600, "regua": 1145, "pnl_compra":    0, "pnl_venda": -573, "par_reguas": -0.500 },
    { "ts": 1786009500, "regua":  586, "pnl_compra": -293, "pnl_venda":    0, "par_reguas": -0.500 },
    { "ts": 1786047300, "regua":  827, "pnl_compra":  540, "pnl_venda": -414, "par_reguas":  0.152 },
    { "ts": 1786085100, "regua":  821, "pnl_compra":  658, "pnl_venda": -364, "par_reguas":  0.358 },
    { "ts": 1786122000, "regua": 1225, "pnl_compra":  524, "pnl_venda": -612, "par_reguas": -0.073 },
    { "ts": 1786159800, "regua":  866, "pnl_compra":    0, "pnl_venda": -433, "par_reguas": -0.500 }
  ]
}
```

---

## Step 3 — Interpreting the results

### Header metrics

| Metric | Value | What it means |
|---|---|---|
| `n_momentos` | 20 | 20 firing moments spaced across the period |
| `soma_pnl` | −$3,747 | Sum of all PnLs (buy + sell) — **negative = the pair bleeds** |
| `soma_par_reguas` | −1.72 | Sum of pairs in régua — negative, but small vs. 20 moments |
| `ev_par_reguas` | −0.086 | Expected value per moment: −0.086 réguas per fire |
| `pares_positivos` | 8 | 8 out of 20 moments (40%) had pair ≥ 0 |

### Reading `por_momento`

Each moment fires **buy AND sell** at the same instant. The `par_reguas` is the sum normalized by the régua:

```
par_reguas = (pnl_compra + pnl_venda) / regua
```

- **`par_reguas = 0`** (moment 3): buy lost exactly what sell gained — a perfect tie.
- **`par_reguas = −0.5`** (moments 1, 4, 12, 13, 15, 16, 20): one side was stopped out and the other didn't execute — the worst case.
- **`par_reguas = +2.28`** (moment 5): the régua was very short (289), so the pair captured disproportionate movement — the best moment.

### The fundamental reading

The expected value is **−0.086 réguas per moment**. Over 20 moments, this accumulates to −1.72 réguas. The pair (buy + sell) **does not survive** — it bleeds ~8.6% of the régua per fire, without fees.

> This **is not a failure of the strategy**. It's the expected result of arbitrary side: without an entry criterion, the adaptive manager can't compensate for the stop cost on both sides. The test says: **your strategy needs an entry factor that adds at least +0.086 réguas per trade to break even**.

---

## Step 4 — Varying parameters

Now let's see which adaptive rules matter most. Each variation changes one parameter, keeping the rest at default:

### Comparison table (20 moments, no fees)

| Variant | `stop_r` | `piramide` | `breakeven` | `reescala_vol` | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|---|---|---|---|
| **Default** | 0.5 | ✓ | ✓ | ✓ | −$3,747 | −0.086 | 8/20 |
| Wide stop | 1.0 | ✓ | ✓ | ✓ | −$3,443 | −0.100 | 9/20 |
| **Tight stop** | **0.25** | ✓ | ✓ | ✓ | **−$2,733** | **−0.017** | **4/20** |
| No pyramid | 0.5 | ✗ | ✓ | ✓ | −$2,437 | −0.071 | 9/20 |
| No breakeven | 0.5 | ✓ | ✗ | ✓ | −$6,675 | −0.237 | 8/20 |
| No rescale | 0.5 | ✓ | ✓ | ✗ | −$2,384 | −0.054 | 9/20 |

### Example: tighter stop (0.25 régua)

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "stop_r": 0.25
  }
}
```

### What we learn

1. **Tight stop (0.25) is the best survivor**: EV of only −0.017 réguas/moment. Cutting losses fast works — the stop is what bleeds most in arbitrary side. But note: pares_positivos drops to 4/20 — the tight stop limits both sides, so few moments capture large movement.

2. **Breakeven is the heaviest rule**: without it, EV worsens from −0.086 to −0.237 (2.8× worse). Breakeven moves the stop to the entry point after moving in favor, limiting damage when the market reverts.

3. **Pyramid and rescale help little**: disabling them even slightly improves EV. Pyramid increases exposure on confirmation, but in arbitrary side, confirmation is noise.

4. **Wide stop (1.0) is worse than default (0.5)**: more régua left on the table when the stop triggers.

---

## Step 5 — The effect of fees

The default test is **without fees** (`fee_pct: 0`) because the doctrine wants to measure the gross floor. But in the real market, fees exist. Let's compare:

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "fee_pct": 0.001
  }
}
```

### Comparison with and without fees (20 moments)

| Configuration | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|
| Default no fee | −$3,747 | −0.086 | 8/20 |
| Default with 0.1% fee | −$24,422 | −0.907 | 1/20 |

The effect is brutal. With 0.1% fee per execution, the EV jumps from −0.086 to **−0.907** réguas per moment — more than 10× worse. Only **1 out of 20** moments survives with fees.

> The survival test with fees is the **reality check**: if your strategy doesn't generate at least +0.9 réguas of edge per trade, it doesn't overcome the execution cost.

---

## Step 6 — More moments for convergence

With 20 moments, variance is high (some moments capture régua 289, others 2031). Let's run 50 moments to see if the EV converges:

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 50
  }
}
```

### Convergence (no fees)

| `momentos` | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|
| 20 | −$3,747 | −0.086 | 8/20 (40%) |
| 50 | −$6,863 | −0.078 | 19/50 (38%) |

The EV converges: from −0.086 to −0.078 réguas per moment. The proportion of positive pairs stabilizes at ~38-40%. More moments give a more stable estimate of the floor, but the conclusion doesn't change: **the pair doesn't survive without fees, and is massacred with fees**.

### Convergence (with 0.1% fee)

| `momentos` | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|
| 20 | −$24,422 | −0.907 | 1/20 (5%) |
| 50 | −$40,776 | −0.892 | 4/50 (8%) |

With fees, the EV converges to approximately **−0.9 réguas per moment** — stable and relentless.

---

## Anatomy of the test

```
                    ┌──────────────────────────────────────────┐
                    │        ct_testar_sobrevivencia            │
                    ├──────────────────────────────────────────┤
                    │                                          │
                    │  N MOMENTS spaced across the period      │
                    │  ┌────┐ ┌────┐ ┌────┐     ┌────┐         │
                    │  │ M1 │ │ M2 │ │ M3 │ ...│ MN │         │
                    │  └─┬──┘ └─┬──┘ └─┬──┘     └─┬──┘         │
                    │    │      │      │           │            │
                    │    ▼      ▼      ▼           ▼            │
                    │  EACH MOMENT fires:                      │
                    │  ┌──────────────────────────┐             │
                    │  │  BUY   (adaptive manager) │             │
                    │  │  SELL  (adaptive manager) │             │
                    │  │  same lib_grupo, params   │             │
                    │  └────────────┬─────────────┘             │
                    │               ▼                            │
                    │  par_reguas = (pnl_buy + pnl_sell) / régua│
                    │               │                            │
                    │               ▼                            │
                    │  AGGREGATE:                                │
                    │  • soma_pnl                               │
                    │  • soma_par_reguas                        │
                    │  • ev_par_reguas (sum / n_momentos)       │
                    │  • pares_positivos (pair ≥ 0)            │
                    └──────────────────────────────────────────┘
```

### Parameters

| Parameter | Default | Description |
|---|---|---|
| `serie` | — | Raw series (`ct://series/...`) to test |
| `momentos` | 20 | Number of spaced fires across the period |
| `stop_r` | 0.5 | Stop in régua (1 régua = average amplitude of `w_vol`) |
| `lib_grupo` | `"grupo"` | Execution lib (fork of `grupo`) |
| `fee_pct` | 0 | Fee per execution (fraction of notional). 0 = no fees |
| `piramide` | true | Pyramid 1→2 lots on confirmation, outside high vol |
| `breakeven` | true | Stop → breakeven after moving `stop_r` in favor |
| `reescala_vol` | true | Trailing rescales with current régua |
| `prazo` | 256 | Maximum position duration in bars (closes at market) |
| `w_vol` | 64 | Régua/vol-fact window |
| `w_tercil` | 480 | Volatility tertile calibration window |
| `ativacao_r` | 1.0 | Trailing activation in réguas in favor |
| `dist_r` | 0.3 | Trailing distance in régua |
| `desde_ts` | — | Start of period (unix seconds). Absent = start of series |
| `ate_ts` | — | End of period (unix seconds). Absent = end of series |

---

## What to do with the result

### Scenario A: `ev_par_reguas ≥ 0` (pair survives)

The adaptive manager survives without an entry criterion. This is **rare and valuable** — it means the stop/take/trailing structure has structural edge (likely capturing mean reversion or autocorrelation). In this case:

- Add an entry factor (indicator, filter, model) that improves the side — each hit becomes pure edge.
- The cost of fees still needs to be overcome, but the floor is positive.

### Scenario B: `ev_par_reguas < 0` (pair bleeds) — the real case

The pair bleeds without fees. This is the most common case and isn't failure — it's **information**. You know exactly how much edge you need to generate:

```
required_edge = |ev_par_reguas|  (in réguas per trade)
```

In our example: **0.086 régua/trade** without fees, **0.907 régua/trade** with 0.1% fees.

Your entry strategy needs to add that to break even. Anything above that is profit.

### Scenario C: `ev_par_reguas << 0` (pair massacred)

If even the manager can't limit the damage (very negative EV, very few positive pairs), the problem is the configuration — not the entry criterion. Before seeking signals, adjust:

1. **Tighten the stop** (0.25 régua instead of 0.5).
2. **Enable breakeven** (the heaviest rule).
3. **Disable pyramid** (in arbitrary side, confirmation is noise).

---

## Next steps

- **Add an entry criterion**: combine the `grupo` lib with indicators to only arm when there's a signal — see [RSI + ADX Filter](./02-rsi-filtro-adx.en.md).
- **Cross-asset**: see how different assets have different survival floors — see [Cross-Asset](./05-cross-asset.en.md).
- **Understand the régua**: the régua is CT Lab's adaptive unit — see [The `grupo` Lib](../docs/05-backtest/05-lib-grupo.en.md).

---

> Back to: [README](../README.en.md) · [Backtest with `grupo`](./03-lib-grupo-backtest.en.md) · [The `grupo` Lib](../docs/05-backtest/05-lib-grupo.en.md)

_Last updated: 2026-08-11_
