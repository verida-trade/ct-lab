# 05 — Cross-Asset: The Floor Varies by Asset

> **Level:** Intermediate · **Premium** · **Prerequisites:** [Survival Test](./04-teste-sobrevivencia.en.md), [The `grupo` Lib](../docs/05-backtest/05-lib-grupo.en.md)

In [example 04](./04-teste-sobrevivencia.en.md) you ran the survival test on BTCUSDT and discovered that the pair bleeds — the adaptive manager doesn't survive without an entry criterion. But is that universal? Do **all assets bleed the same way**?

The answer is no. The survival floor **varies dramatically** across assets. Some assets have such strong mean reversion that the pair survives even without fees — and sometimes even **with fees**. Others bleed so much that even the best manager can't save them.

In this example, you will:

1. **Fetch data for 5 assets** — BTC, ETH, SOL, BNB, and XRP at 15m.
2. **Run the survival test** on each one, with the same parameters.
3. **Compare the floors** — see which assets survive and which bleed.
4. **Add fees** and see which assets still survive.
5. **Understand why** — what makes an asset a survivor.
6. **Convergence** with 50 moments to validate the reading.

---

## The problem

You're deciding which assets to trade. Your strategy uses the `grupo` lib with 0.5-régua stop, breakeven, and trailing — the standard adaptive manager from the doctrine. Before spending months looking for an entry signal, you want to know: **which asset gives your strategy the highest survival floor?**

The cross-asset survival test answers this in minutes. You run the same test across multiple assets and compare the `ev_par_reguas` — the expected value per moment in réguas. Assets with positive EV have surplus. Assets with very negative EV need more edge to compensate.

> The régua is CT Lab's adaptive unit: 1 régua = average amplitude of the `w_vol` window (64 bars by default). The 0.5-régua stop adjusts to the current volatility of **each asset** — for BTC, 1 régua might be $1,200; for XRP, it might be $0.01. This normalization lets you compare assets with completely different prices.

---

## Step 1 — Fetch data for 5 assets

```
Fetch BTCUSDT, ETHUSDT, SOLUSDT, BNBUSDT, and XRPUSDT at 15m from Binance.
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "ETHUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "SOLUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "BNBUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "XRPUSDT", "interval": "15m" }
}
```

> Each series becomes available at `ct://series/binance/<SYMBOL>/15m`.

---

## Step 2 — Run the test on each asset (20 moments, no fees)

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": { "serie": "ct://series/binance/BTCUSDT/15m" }
}
```

Repeat for each asset, changing the `symbol` in the URI.

### Comparative results (20 moments, no fees)

| Asset | Approx. price | Approx. régua | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|---|---|
| **BTC** | ~$110k | ~$1,200 | −$3,807 | **−0.095** | 7/20 (35%) |
| **ETH** | ~$3,500 | ~$45 | −$263 | **−0.449** | 4/20 (20%) |
| **SOL** | ~$180 | ~$2 | +$9 | **+0.254** | 16/20 (80%) |
| **BNB** | ~$550 | ~$6 | +$246 | **+2.054** | 20/20 (100%) |
| **XRP** | ~$0.50 | ~$0.01 | −$0.19 | **−0.337** | 4/20 (20%) |

### The reading

The results reveal **three classes of assets**:

1. **BNB — absolute survivor**: EV of +2.05 réguas per moment, 20/20 positive pairs. The adaptive manager doesn't just survive — it **profits** without an entry criterion. This suggests very strong mean reversion in the tested period.

2. **SOL — marginal survivor**: EV of +0.254, 16/20 positive. The pair survives, but with thin margin. Any real cost (slippage, spread) can eat that margin.

3. **BTC, ETH, XRP — non-survivors**: All have negative EV. BTC is the "least bad" (−0.095), XRP and ETH are the worst (−0.337 and −0.449). These assets need an entry factor that adds edge to compensate.

> **Note**: these results are specific to the sampled period (1000 15m candles) and the default manager configuration. They are not universal — an asset's floor changes with market regime. The value of the cross-asset test lies in the **relative comparison**: which asset gives your strategy the most room.

---

## Step 3 — The effect of fees per asset

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": { "serie": "ct://series/binance/BNBUSDT/15m", "fee_pct": 0.001 }
}
```

### Comparative results (20 moments, with 0.1% fee)

| Asset | `soma_pnl` (no fee) | `soma_pnl` (with fee) | `ev_par_reguas` (with fee) | `pares_positivos` (with fee) |
|---|---|---|---|---|
| **BTC** | −$3,807 | −$24,609 | **−0.922** | 1/20 (5%) |
| **ETH** | −$263 | −$451 | **−0.832** | 0/20 (0%) |
| **SOL** | +$9 | −$15 | **−0.361** | 10/20 (50%) |
| **BNB** | +$246 | +$186 | **+1.580** | 19/20 (95%) |
| **XRP** | −$0.19 | −$0.28 | **−0.495** | 1/20 (5%) |

### The reading with fees

The picture changes drastically:

- **BNB** is the **only** asset that survives with fees: EV of +1.58 réguas, 19/20 positive pairs. The structural edge is so strong that it absorbs 0.1% fee per execution and still has surplus.
- **SOL** lost its floor: dropped from +0.254 to −0.361. The margin was thin and fees ate it all.
- **BTC, ETH, XRP** worsen proportionally. ETH is the most extreme case: **0/20** positive pairs with fees — not a single moment survives.

> The test with fees is the **final filter**. Assets that survive without fees but die with fees (like SOL) need an entry factor that adds at least +0.361 réguas of edge. BNB already has positive net edge — any entry factor is pure profit.

---

## Step 4 — Inside: BNB vs BTC

Let's look at `por_momento` to understand **why** BNB survives and BTC doesn't.

### BNB — the survivor (20 moments, no fees)

| Moment | Régua | PnL buy | PnL sell | `par_reguas` |
|---|---|---|---|---|
| 1 | 9 | +8 | −5 | +0.31 |
| 2 | 9 | +22 | −5 | +1.81 |
| 3 | 9 | +21 | −5 | +1.77 |
| 4 | 9 | +14 | −5 | +0.99 |
| 5 | 9 | +13 | −5 | +0.88 |
| ... | ... | ... | ... | ... |
| 12 | 7 | +24 | −3 | +3.23 |
| 13 | 5 | +25 | −2 | +5.12 |
| 14 | 4 | +20 | −2 | +4.51 |
| ... | ... | ... | ... | ... |
| 20 | 6 | +10 | −3 | +1.14 |

**Pattern**: In BNB, the buy side **consistently captures more than the sell side loses**. The régua is small (4-9), the stop is tight, and the asset has strong mean reversion — the price rises and falls within the channel, and the manager captures movement from both sides. `par_reguas` is positive across **all 20 moments**.

### BTC — the non-survivor (20 moments, no fees)

| Moment | Régua | PnL buy | PnL sell | `par_reguas` |
|---|---|---|---|---|
| 1 | 1233 | 0 | −617 | −0.50 |
| 2 | 1548 | −774 | +280 | −0.32 |
| 3 | 2031 | −329 | +329 | 0.00 |
| 4 | 1383 | −680 | 0 | −0.49 |
| 5 | 289 | −145 | +803 | +2.28 |
| ... | ... | ... | ... | ... |
| 12 | 922 | 0 | −461 | −0.50 |
| 13 | 934 | −467 | 0 | −0.50 |
| ... | ... | ... | ... | ... |
| 20 | 866 | 0 | −433 | −0.50 |

**Pattern**: In BTC, the régua is enormous (289-2031) and `par_reguas` frequently hits **−0.50** — the pure stop value. This means: one side got stopped out and the other didn't execute at all. BTC's directional movement in the period was large enough that, in many moments, the manager couldn't capture both sides — one side takes the stop and the other sits idle.

### The structural difference

| Characteristic | BNB | BTC |
|---|---|---|
| Typical régua | 4–9 | 600–2000 |
| Dominant `par_reguas` | +0.3 to +5.0 (positive) | −0.50 (pure stop) |
| Movement | Channel (mean reversion) | Directional trend |
| Manager captures | Both sides | One side takes stop |

BNB has **short régua and channel** — the adaptive manager thrives because the price reverts within the range. BTC has **long régua and trend** — the manager suffers because the price runs away and stops out one side.

---

## Step 5 — Convergence with 50 moments

With 20 moments, variance is high. Let's run 50 moments to confirm the results aren't noise:

### Convergence (50 moments, no fees)

| Asset | `soma_pnl` (20) | `soma_pnl` (50) | `ev_par_reguas` (20) | `ev_par_reguas` (50) | `pares_positivos` (50) |
|---|---|---|---|---|---|
| BTC | −$3,807 | −$8,769 | −0.095 | −0.128 | 18/50 (36%) |
| ETH | −$263 | −$499 | −0.449 | −0.352 | 15/50 (30%) |
| SOL | +$9 | +$26 | +0.254 | +0.296 | 42/50 (84%) |
| BNB | +$246 | +$614 | +2.054 | +2.078 | 50/50 (100%) |
| XRP | −$0.19 | −$0.51 | −0.337 | −0.363 | 7/50 (14%) |

### Convergence (50 moments, with 0.1% fee)

| Asset | `soma_pnl` (50) | `ev_par_reguas` (50) | `pares_positivos` (50) |
|---|---|---|---|
| BTC | −$46,943 | −1.093 | 4/50 (8%) |
| ETH | −$922 | −0.680 | 0/50 (0%) |
| SOL | −$45 | −0.408 | 24/50 (48%) |
| BNB | +$467 | +1.602 | 49/50 (98%) |
| XRP | −$0.73 | −0.521 | 0/50 (0%) |

### What convergence confirms

1. **BNB is structurally a survivor**: 50/50 positive pairs without fees, 49/50 with fees. The EV converges to +2.08 without fees and +1.60 with fees — stable and robust.

2. **SOL confirms the marginal floor**: EV +0.30 without fees (slightly better than 20 moments), but −0.41 with fees. The no-fee margin is real but too thin to absorb execution cost.

3. **BTC worsens slightly with more moments**: EV from −0.095 to −0.128. The directional trend is more consistent than 20 moments suggested.

4. **ETH and XRP are the worst**: ETH has 0/50 with fees, XRP too. Both need substantial edge before being tradable.

---

## Step 6 — What makes an asset a survivor?

The cross-asset test reveals that survival depends on the **microstructural character of the asset**:

| Characteristic | Survivor (BNB, SOL) | Non-survivor (BTC, ETH, XRP) |
|---|---|---|
| **Régua** | Short (4-9 in BNB) | Long (600-2000 in BTC) |
| **Movement** | Channel / mean reversion | Directional trend |
| **Autocorrelation** | Negative (reversion) | Positive (momentum) |
| **Adaptive stop** | Flags fast, captures reversion | Wide stop, loses in direction |
| **Arbitrary side** | Wins — both sides capture | Loses — one side takes stop |

### Why does BNB have short régua?

The régua is the average amplitude of the `w_vol` window (64 bars). BNB at 15m has low relative volatility in the period — $4-$9 moves per bar. With 0.5-régua stop, the manager operates in a $2-$5 channel. When the price reverts (and BNB has strong 15m reversion), the opposite side captures quickly.

### Why does BTC have long régua?

BTC moves $600-$2000 per bar at 15m. With 0.5-régua stop, the manager needs $300-$1000 moves to capture. In strong trend periods, the price runs away before reverting — one side is stopped and the other doesn't have time to execute.

> **It's not that BTC is "worse" than BNB** — it's that the **adaptive manager structure** (stop + breakeven + trailing) favors assets with mean reversion. In a trending asset, the manager needs an **entry factor** that filters momentum to avoid trading against the trend.

---

## How to use the cross-asset result

### Asset selector

Before developing a strategy, run the survival test on a basket of assets. Pick the ones with **EV closest to zero or positive** — these give your strategy the most room.

```
ranking = sort_assets_by(ev_par_reguas)
# BNB (+2.05) > SOL (+0.25) > BTC (−0.095) > XRP (−0.337) > ETH (−0.449)
```

### Edge sizing

For each asset, `|ev_par_reguas|` tells you **how much edge you need to generate**:

| Asset | Required edge (no fee) | Required edge (with 0.1% fee) |
|---|---|---|
| BNB | 0 (already positive) | 0 (already positive) |
| SOL | 0 (already positive) | 0.361 réguas/trade |
| BTC | 0.095 réguas/trade | 0.922 réguas/trade |
| XRP | 0.337 réguas/trade | 0.495 réguas/trade |
| ETH | 0.449 réguas/trade | 0.832 réguas/trade |

- **BNB**: any entry factor becomes net profit — even with fees.
- **ETH**: needs to generate 0.83 réguas of edge per trade just to break even. That's a lot.
- **BTC**: needs 0.92 réguas with fees — hard, but feasible with a good model.

### Caveats

1. **Period matters**: results are specific to the 1000 sampled candles. In a different period (bull vs bear market), the ranking may change.
2. **Timeframe matters**: BTC at 1h might have shorter régua (less noise) and better survival.
3. **Regime matters**: BNB surviving in a channel doesn't mean it survives in a trend. Run the test periodically.
4. **The test doesn't replace backtesting**: the survival test measures the **floor** — the lower bound. Your real strategy (with entry criterion) should do better than the floor.

---

## Next steps

- **Strategy with entry criterion**: combine the manager with indicators to only trade survivors — see [RSI + ADX Filter](./02-rsi-filtro-adx.en.md).
- **Test other timeframes**: run the test at 1h or 4h and see if the ranking changes — see [Survival Test](./04-teste-sobrevivencia.en.md).
- **ML model**: train a model to predict side on assets with negative floor — see [GBDT Model](./06-modelo-gbdt.en.md).

---

> Back to: [README](../README.en.md) · [Survival Test](./04-teste-sobrevivencia.en.md) · [The `grupo` Lib](../docs/05-backtest/05-lib-grupo.en.md)

_Last updated: 2026-08-11_
