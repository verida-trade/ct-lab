# 09 — Regime: When the Market Changes Personality

> **Level:** Advanced · **Premium** · **Prerequisites:** [Microstructure TFI](./08-tfi-backtest.en.md), [GBDT Model](./06-modelo-gbdt.en.md)

The GBDT from example 06 generated +$12,447 net profit with fees. The BOP from example 08 had positive edge without fees (+$398) but was destroyed by fees (−$16,178). The question that won't go away: **do those results hold across the entire period, or only half of it?**

Every market alternates between **regimes** — phases with distinct statistical properties. In a trending regime, price persists in one direction: what went up yesterday tends to go up today. In a sideways regime, price reverts: what went up yesterday tends to fall today. If you don't know which regime you're in, your strategy may be perfectly adapted to the wrong regime.

In this example you'll:
1. **Measure the structure** of the price path — variance ratio and kurtosis by scale.
2. **Detect regime** using ADX (trending) vs RSI (sideways).
3. **Compare strategies** in each regime — trend-following vs mean reversion.
4. **Combine regime + signal** in a backtest with dynamic regime filtering.

---

## The problem

A strategy is a bet on a statistical property of price:

- **Trend-following** bets that `Var(pT) / T > Var(p1)` — price persists. Works when VR > 1.
- **Mean reversion** bets that `Var(pT) / T < Var(p1)` — price reverts. Works when VR < 1.

If you run trend-following in a sideways market (VR < 1), you buy at the top and sell at the bottom — exactly the opposite of what you should do. The problem is that **the regime changes over time** and you need to detect when.

> **Variance Ratio (VR)** measures whether price is a random walk. VR = 1 means random walk (no memory). VR > 1 means persistence (trending). VR < 1 means anti-persistence (reverting). It's the most fundamental regime metric.

---

## Step 1 — Measure price path structure

The `ct_medir_estrutura` tool computes variance ratio and kurtosis by scale (1, 4, 8, ..., 512 bars), subdivided by volatility tercile and temporal block:

```json
{
  "name": "ct_medir_estrutura",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m"
  }
}
```

### 1.1 — Overall variance ratio by scale

| Scale (τ) | VR | Kurtosis | Interpretation |
|---|---|---|---|
| 1 | 1.00 | 6.86 | (reference) |
| 4 | 0.94 | 8.24 | Slight antipersistence |
| 8 | 0.96 | 5.18 | ≈ random walk |
| 16 | 0.99 | 2.27 | ≈ random walk |
| 32 | 0.97 | 1.34 | ≈ random walk |
| 64 | 0.74 | 1.63 | Strong antipersistence |
| 128 | 0.58 | 0.19 | Strong antipersistence |
| 256 | 0.42 | −0.32 | Very strong antipersistence |
| 512 | 0.29 | −0.72 | Extreme antipersistence |

> **At short scale (τ=1–32), price is nearly a random walk** (VR ≈ 1). At long scale (τ=64–512), **strong antipersistence** (VR < 1). This means: in the short term the price has no preferred direction, but over long windows it **reverts** — what goes up a lot tends to come down. It's a market that **lacks sustained trends**.

### 1.2 — VR by volatility tercile

| Scale | Low vol | Mid vol | High vol |
|---|---|---|---|
| 4 | 0.80 | 0.73 | 1.22 |
| 8 | 0.77 | 0.68 | 1.34 |
| 16 | 0.85 | 0.67 | 1.23 |
| 32 | 0.98 | 1.02 | 0.81 |
| 64 | 0.58 | 1.14 | 0.46 |

> **In high volatility, VR > 1 at short scales** (τ=4–16): the price persists — there's directional movement with volatility. In low volatility, VR < 1: the price oscillates sideways. The conclusion: **volatility regime determines persistence**.

### 1.3 — VR by temporal block (8 blocks of ~36h each)

| Period | VR(τ=4) | VR(τ=16) | Regime |
|---|---|---|---|
| Block 1 (Aug 10–11) | 1.26 | 0.93 | Short-term trend |
| Block 2 (Aug 11–13) | 0.80 | 1.19 | Sideways → trend |
| Block 3 (Aug 13–14) | 1.07 | 1.32 | Trend |
| Block 4 (Aug 14–15) | 0.65 | 0.29 | Strong sideways |
| Block 5 (Aug 15–17) | 0.97 | 1.01 | Random walk |
| Block 6 (Aug 17–18) | 0.53 | 0.35 | Strong sideways |
| Block 7 (Aug 18–20) | 0.71 | 0.98 | Sideways |
| Block 8 (Aug 20–21) | 1.05 | 1.34 | Trend |

> **The regime changes from block to block.** Block 3 was trending (VR > 1), block 4 was sideways (VR = 0.29), block 8 went back to trending. A fixed strategy would be destroyed half the time.

---

## Step 2 — Detect regime with ADX

Variance Ratio is a post-hoc analysis — you can't use it in real time because it needs hundreds of bars. **ADX** (Average Directional Index) is a real-time proxy: it measures trend strength over the last 14 candles.

### 2.1 — Materialize ADX

```json
{
  "name": "adx",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 14, "name": "btc_adx_regime" }
}
```

### 2.2 — Build regime features + BOP + RSI in a single pipeline

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "r09_bop_adx_rsi",
    "output": "$features",
    "steps": [
      { "op": "bop", "id": "bop", "source": "$anchor" },
      { "op": "adx", "id": "adx", "source": "$anchor", "period": 14 },
      { "op": "rsi", "id": "rsi", "source": "$anchor", "column": "close", "period": 14 },
      {
        "op": "compose",
        "id": "features",
        "columns": [
          { "as_column": "bop", "source": "$bop", "source_column": "bop" },
          { "as_column": "adx", "source": "$adx", "source_column": "adx" },
          { "as_column": "plus_di", "source": "$adx", "source_column": "plus_di" },
          { "as_column": "minus_di", "source": "$adx", "source_column": "minus_di" },
          { "as_column": "rsi", "source": "$rsi", "source_column": "rsi" }
        ]
      }
    ]
  }
}
```

> The series `ct://derived/r09_bop_adx_rsi` contains BOP, ADX, +DI, −DI, and RSI at 15m — all aligned at the same timestamp.

---

## Step 3 — Backtests by regime

Now you run 4 strategies and compare. All use the same composite series `r09_bop_adx_rsi` as indicators:

### 3.1 — BOP base (no regime filter)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"bop\"][0] > 0.2 { comprado(1.0) } else if ind[\"bop\"][0] < -0.2 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_bop_base_fee"
  }
}
```

### 3.2 — BOP filtered by ADX > 25 (trending only)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"bop\"][0] > 0.2 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"bop\"][0] < -0.2 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_bop_trend_fee"
  }
}
```

### 3.3 — BOP filtered by ADX < 20 (sideways only)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"bop\"][0] > 0.2 && ind[\"adx\"][0] < 20.0 { comprado(1.0) } else if ind[\"bop\"][0] < -0.2 && ind[\"adx\"][0] < 20.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_bop_choppy_fee"
  }
}
```

### 3.4 — RSI reversal in sideways (ADX < 20, RSI < 30 or > 70)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_rsi_chop_fee"
  }
}
```

---

## Step 4 — Results

### Without fees (gross edge)

| Strategy | Regime | Trades | PnL | Win% | PF | Sharpe | Calmar | Max DD |
|---|---|---|---|---|---|---|---|---|
| BOP base | All | 866 | −$3,649 | 38.3% | 0.88 | −0.022 | −2.47 | 40.4% |
| BOP ADX>25 | Trend | 418 | −$2,238 | 36.1% | 0.84 | −0.021 | −3.68 | 27.1% |
| BOP ADX<20 | Sideways | 284 | +$804 | 43.0% | 1.08 | 0.011 | 19.6 | 19.8% |
| RSI reversal (ADX<20) | Sideways | 47 | +$1,334 | 76.6% | 1.66 | 0.027 | 133.3 | 9.0% |
| RSI trend (ADX>25) | Trend | 174 | +$246 | 40.8% | 1.03 | 0.005 | 5.01 | 12.9% |
| Buy & Hold | — | 0 | −$296 | — | — | 0.003 | −1.59 | 28.9% |

### With 0.1% fees

| Strategy | Regime | Trades | PnL | Win% | PF | Sharpe | Calmar |
|---|---|---|---|---|---|---|---|
| BOP base | All | 866 | −$114,880 | 7.2% | 0.074 | 0.025 | −0.087 |
| BOP ADX>25 | Trend | 418 | −$55,875 | 7.9% | 0.062 | 0.027 | −0.179 |
| BOP ADX<20 | Sideways | 284 | −$35,715 | 8.5% | 0.107 | 0.001 | −0.280 |
| RSI reversal (ADX<20) | Sideways | 47 | −$4,707 | 8.5% | 0.174 | −0.086 | −2.01 |
| RSI trend (ADX>25) | Trend | 174 | −$22,098 | 13.2% | 0.101 | 0.004 | −0.453 |
| Buy & Hold | — | 0 | −$296 | — | — | 0.003 | −1.59 |

### Analysis

**Without fees, the regime filter transforms the strategy:**

1. **BOP in sideways (ADX < 20)**: PnL +$804, profit factor 1.08, Calmar 19.6 — the best BOP. The filter removes trending trades (which are losers for BOP in this market) and keeps the ones that work.

2. **RSI reversal in sideways**: the best strategy of all. Only 47 trades, win rate 76.6%, profit factor 1.66, Calmar 133, max drawdown 9%. Expectancy per trade is +$28.4 — each trade adds 0.28% to capital.

3. **BOP without filter**: PnL −$3,649 — a loser because half the trades happen in trends, where BOP is counterproductive.

4. **BOP in trend (ADX > 25)**: PnL −$2,238 — worse than no filter. BOP is intrabar momentum, not trend prediction; filtering by trend doesn't help.

> **The key finding: this market is antipersistent at long scale** (VR < 1 at τ=64–512). Reversion strategies outperform trend-following. The ADX < 20 filter captures exactly the sideways periods where reversion dominates.

### With fees, everything is destroyed

RSI reversal has only 47 trades — far fewer than the 866 of base BOP — but fees ($6,041) still exceed gross PnL ($1,334). The structural problem is the same as example 08: the gross edge isn't large enough to overcome execution costs.

> **But the regime lesson survives**: fees destroy the PnL, but the strategy **ranking** is the same. RSI reversal in sideways is better than BOP in trend. If you add RSI reversal as an **entry filter** to the GBDT (example 06) — which already overcame fees — the regime filter can improve the model.

---

## Step 5 — Observations on Variance Ratio

The VR-by-temporal-block table (Step 1.3) reveals something important: **the regime changes not only between periods, but also between scales**:

- At τ=4 (1 hour), the market alternates between VR > 1 (block 3, trend) and VR < 1 (block 4, sideways) — transient regimes.
- At τ=512 (~5 days), VR = 0.29 consistently — **strong reversion in all windows**.

This means short-duration strategies (~1h) need to detect regime in real time (ADX, Bollinger width). Long-duration strategies (>1 day) can assume reversion.

> **Negative kurtosis** (platykurtic at τ=512) means the long-horizon return distribution is more "flattened" than normal — fewer extreme values. The market converts extremes into averages. This favors mean reversion.

---

## When to use each strategy

| Regime | ADX | VR | Strategy | Indicator |
|---|---|---|---|---|
| Strong trend | > 25 | > 1 | Trend-following | Directional BOP, RSI > 50 |
| Sideways | < 20 | < 1 | Mean reversion | RSI extremes (< 30 / > 70) |
| Transition | 20–25 | ≈ 1 | Don't trade | — |

> **Regime filtering doesn't add edge — it removes anti-edge.** Instead of trying to win more, you try not to trade in moments when your strategy systematically loses.

---

## Next steps

- **Regime + GBDT**: add ADX and BB width as features in the example 06 pipeline — the model learns to adjust its predictions by regime automatically.
- **Fork the doctrine**: use ADX < 20 as a filter for the adaptive manager — only enter when the market is sideways and reversion dominates — see [Fork Doctrine](./11-fork-doutrina.en.md).
- **Full ML pipeline**: automate regime detection in an ML node (regime classifier as a pre-step to the model) — see [Full ML Pipeline](./10-esteira-ml-completa.en.md).
- **Microstructure + regime**: combine TFI/OBI (example 08) with ADX filter — enter only when microstructure and regime converge.

---

> Back to: [README](../README.md) · [Microstructure TFI](./08-tfi-backtest.en.md) · [GBDT Model](./06-modelo-gbdt.en.md) · [Full ML Pipeline](./10-esteira-ml-completa.en.md)

_Last updated: 2026-08-12_
