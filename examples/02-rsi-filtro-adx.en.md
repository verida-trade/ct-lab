# 02 — RSI with ADX Filter: Trend and Momentum

> **Level:** Intermediate · **Time:** ~20 min · **Prerequisites:** [Example 01 — SMA Crossover](./01-cruzamento-sma.en.md)

RSI (Relative Strength Index) is one of the most popular indicators in trading — but it suffers from a problem: it fires signals in choppy markets. ADX (Average Directional Index) measures trend strength, not direction. Combining the two seems promising: only trade RSI signals when ADX confirms a strong trend.

In this example, you'll discover that **the filter reduces fees — but doesn't create edge**.

---

## What are RSI and ADX?

| Indicator | What it measures | Range | Interpretation |
|-----------|-----------------|-------|----------------|
| **RSI** | Momentum (speed of price changes) | 0–100 | < 30 = oversold · > 70 = overbought |
| **ADX** | Trend strength (direction-independent) | 0–100 | > 25 = strong trend · < 20 = choppy |

### Why combine them?

- **RSI alone:** generates reversal signals in any market — including choppy ones, where those signals are false.
- **ADX alone:** doesn't tell you to buy or sell — only whether the market is trending.
- **Combination:** use RSI as the trigger and ADX as the filter. Only trade when RSI is at extremes **AND** ADX confirms a strong trend.

The idea is to reduce whipsaws (false signals in choppy markets). The question is: does that reduction outweigh the cost of losing good signals?

---

## Step 1 — Fetch the series

**Chat (AI):**
> Fetch BTCUSDT 15m from Binance

**MCP tool call:**
```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

The series is available at `ct://series/binance/BTCUSDT/15m` with 1,724 candles (~18 days).

---

## Step 2 — Backtest pure RSI (no fee)

Before adding the filter, let's see RSI alone.

**Chat (AI):**
> Run a backtest: buy when RSI < 30, sell when RSI > 70, else flat. Capital 1000, no fee.

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "rsi_puro_nofee"
  }
}
```

**Real return (summary):**
```json
{
  "num_trades": 135,
  "pnl_total": 1496.02,
  "pnl_bruto": 1496.02,
  "fees_totais": 0,
  "sharpe": 0.033,
  "sortino": 0.049,
  "win_rate": 0.7333,
  "profit_factor": 1.228,
  "avg_win": 81.29,
  "avg_loss": -181.99,
  "payoff_ratio": 0.447,
  "drawdown_max": 70.07,
  "exposicao": 0.2123,
  "num_long": 56,
  "num_short": 79,
  "max_wins_seguidos": 15,
  "max_perdas_seguidas": 3
}
```

### The allure of 135 trades

Without fees, pure RSI is profitable: **+$1,496** with 73.3% win rate. Looks excellent.

But look closer:

- **Payoff ratio = 0.447:** average win ($81) is less than half the average loss (-$182). The strategy wins frequently but loses big.
- **70% drawdown:** at some point the equity dropped 70% from peak.
- **135 trades in 1,724 candles:** roughly one trade every 13 candles. That's high turnover.

The critical question: how much of that $1,496 survives transaction costs?

---

## Step 3 — Pure RSI with 0.1% fee

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_puro_fee"
  }
}
```

**Real return (summary):**
```json
{
  "num_trades": 135,
  "pnl_total": -15833.95,
  "pnl_bruto": 1496.02,
  "fees_totais": 17329.97,
  "sharpe": 0.040,
  "win_rate": 0.0963,
  "profit_factor": 0.099
}
```

The gross edge of **+$1,496** is annihilated by **$17,330** in fees. The fee-to-edge ratio is **11.6:1**.

With 135 trades at ~$128 round-trip:

```
$128 × 135 = $17,280 ≈ $17,330 ✓
```

**Pure RSI is the worst case for fees:** many trades (135), each with small edge. Transaction cost is 11× larger than the entire gross edge.

---

## Step 4 — The ADX filter

The theory: if we filter RSI signals in non-trending markets (low ADX), we reduce the number of trades and whipsaw.

### The ADX-as-recipe gotcha

ADX returns a map with multiple columns (`adx`, `plus_di`, `minus_di`), not a single series. Therefore, it **does NOT work as `indicadores_receitas`** — it must be materialized via a pipeline.

**MCP tool call (pipeline):**
```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_adx_rsi",
    "output": "$concat",
    "steps": [
      { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
      { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
      {
        "id": "concat",
        "op": "compose",
        "columns": [
          { "source": "$adx", "source_column": "adx", "as_column": "adx" },
          { "source": "$rsi", "source_column": "rsi", "as_column": "rsi" }
        ]
      }
    ]
  }
}
```

**Real return:**
```json
{
  "uri": "ct://derived/btc_adx_rsi",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1724,
  "first_ts": 1784960100,
  "last_ts": 1786510800,
  "steps_executed": 3
}
```

The pipeline produces a derived series with `adx` and `rsi` columns aligned bar by bar. The backtest uses `indicadores` (URI) instead of `indicadores_receitas`.

---

## Step 5 — Backtest RSI+ADX (no fee)

**Strategy:** buy when RSI < 30 **AND** ADX > 25; sell when RSI > 70 **AND** ADX > 25.

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_adx_rsi",
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "rsi_adx_nofee"
  }
}
```

**Real return (summary):**
```json
{
  "num_trades": 72,
  "pnl_total": -146.79,
  "pnl_bruto": -146.79,
  "fees_totais": 0,
  "sharpe": 0.026,
  "win_rate": 0.6389,
  "profit_factor": 0.963,
  "drawdown_max": 79.06,
  "exposicao": 0.1288
}
```

The ADX filter reduced trades from 135 to 72 (–47%), but the gross PnL went from +$1,496 to **–$146.79**. The filter destroyed more edge than whipsaw.

---

## Step 6 — Backtest RSI+ADX (with 0.1% fee)

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_adx_rsi",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_adx_fee"
  }
}
```

**Real return (summary):**
```json
{
  "num_trades": 72,
  "pnl_total": -9380.84,
  "pnl_bruto": -146.79,
  "fees_totais": 9234.05,
  "sharpe": -0.013,
  "win_rate": 0.0972,
  "profit_factor": 0.066,
  "drawdown_max": 9.49,
  "exposicao": 0.1288,
  "avg_win": 94.43,
  "avg_loss": -154.49,
  "payoff_ratio": 0.611
}
```

With fees, total losses are $9,381 — less than pure RSI with fees ($15,834) because there are fewer trades, but still catastrophic.

---

## Step 7 — ADX threshold variations

Would a different ADX threshold work better? Let's test 20 (more permissive) and 30 (more restrictive).

### ADX > 20 (more permissive, more trades)

**Strategy:** `ind["adx"][0] > 20.0` instead of `> 25.0`

**Real return (summary):**
```json
{
  "num_trades": 95,
  "pnl_total": -12101.97,
  "pnl_bruto": 88.40,
  "fees_totais": 12190.37,
  "sharpe": 0.023,
  "win_rate": 0.0947,
  "profit_factor": 0.059,
  "drawdown_max": 12.21
}
```

### ADX > 30 (more restrictive, fewer trades)

**Strategy:** `ind["adx"][0] > 30.0`

**Real return (summary):**
```json
{
  "num_trades": 53,
  "pnl_total": -5902.16,
  "pnl_bruto": 880.48,
  "fees_totais": 6782.64,
  "sharpe": 0.019,
  "win_rate": 0.1698,
  "profit_factor": 0.120,
  "drawdown_max": 6.01
}
```

---

## Comparative table — 7 strategies

| # | Strategy | Trades | PnL Total | PnL Gross | Fees | Sharpe | Win% | PF | Payoff | DD Max |
|---|----------|--------|-----------|-----------|------|--------|------|----|--------|--------|
| 1 | Pure RSI (no fee) | 135 | +$1,496 | +$1,496 | $0 | 0.033 | 73.3% | 1.228 | 0.447 | 70.1% |
| 2 | Pure RSI (0.1% fee) | 135 | -$15,834 | +$1,496 | $17,330 | 0.040 | 9.6% | 0.099 | — | — |
| 3 | RSI+ADX>25 (no fee) | 72 | -$147 | -$147 | $0 | 0.026 | 63.9% | 0.963 | — | 79.1% |
| 4 | RSI+ADX>25 (0.1% fee) | 72 | -$9,381 | -$147 | $9,234 | -0.013 | 9.7% | 0.066 | 0.611 | 9.5% |
| 5 | RSI+ADX>20 (0.1% fee) | 95 | -$12,102 | +$88 | $12,190 | 0.023 | 9.5% | 0.059 | — | 12.2% |
| 6 | RSI+ADX>30 (0.1% fee) | 53 | -$5,902 | +$880 | $6,783 | 0.019 | 17.0% | 0.120 | — | 6.0% |
| 7 | Buy & Hold | 0 | -$282 | — | — | -0.024 | — | — | — | 1.3% |

### Analysis by column

**Trades:** the ADX filter reduces from 135 (pure RSI) to 72 (ADX>25), 95 (ADX>20), and 53 (ADX>30). Fewer trades = fewer fees.

**PnL Gross:** here's the problem. Pure RSI has +$1,496 gross, but the ADX>25 filter reduces it to **–$147**. The filter destroyed nearly all the gross edge. ADX>30 is better (+$880 gross) but still below pure RSI.

**Fees:** ADX>30 pays $6,783 in fees (vs $17,330 for pure RSI). A 61% reduction in fees — but since gross edge fell from $1,496 to $880, the edge-to-fee ratio actually worsened.

**Best variant with fees:** ADX>30 is the least bad ($5,902 loss vs $15,834 for pure), because it has fewer trades and positive gross edge.

---

## The filter dilemma

The result reveals the fundamental tension of filtering:

```
                    Pure RSI          ADX>25            ADX>30
Trades:               135               72                53
Gross edge:        +$1,496           -$147            +$880
Fees (0.1%):       $17,330          $9,234            $6,783

Edge/trade:          $11.08           -$2.04            $16.61
Fee/trade:           $128.37          $128.25           $128.00
Edge/fee ratio:      0.086            -0.016            0.130
```

The lessons:

1. **The filter reduced trades** (from 135 to 53) — which is good for fees.
2. **But the filter also reduced gross edge** (from $1,496 to $880 in the best case) — which is bad for PnL.
3. **The edge-to-fee ratio improved** (from 0.086 to 0.130 with ADX>30) but is still an order of magnitude below 1.0.
4. **A filter doesn't create edge** — it only removes signals. If some of those signals were good, you lost edge.

The right question isn't "does the filter improve win rate?" (yes, from 9.6% to 17.0% with fees) but "does the filter improve the edge-to-fee ratio enough to make the strategy profitable?" (no — 0.13 ≪ 1.0).

---

## Variations to experiment with

| # | Variation | How | Hypothesis |
|---|-----------|-----|------------|
| 1 | **Tighter RSI** | Oversold 25, overbought 75 | Fewer signals, less whipsaw |
| 2 | **RSI on EMA** | Change `rsi(close, 14)` to `rsi(ema(close, 14), 14)` | Smooths the RSI, reduces jaggedness |
| 3 | **ADX with DI direction** | Only buy if DI+ > DI− AND ADX > 25 | Filters reversals against the trend |
| 4 | **ATR for sizing** | `atr(high, low, close, 14)` and adjust lot by volatility | Reduces risk in high-volatility periods |
| 5 | **Higher timeframe** | Re-fetch with `interval: "1h"` | Fewer trades, larger edge per trade |
| 6 | **RSI with ATR exit** | Exit after X bars instead of RSI > 70 | Avoids waiting for overbought to exit |
| 7 | **Combine with SMA** | Only trade if close > SMA 200 (long-term trend filter) | Aligns with macro trend |

---

## Next steps

- [Example 03 — Backtest with `grupo` lib](./03-lib-grupo-backtest.en.md) — Sophisticated execution with scaled entries and OCO stops.
- [Example 04 — Survival test](./04-teste-sobrevivencia.en.md) — The random-side floor.
- [Example 08 — Microstructure: TFI](./08-tfi-backtest.en.md) — Order flow indicators.
- [Example 12 — Rhai inline](./12-rhai-inline-backtest.en.md) — 8 classic strategies compared.

---

> **Lesson:** A filter that reduces trades without improving the edge-to-fee ratio is just cosmetic. ADX cut the whipsaws — but it also cut the trades that generated the edge. Filtering is easy; filtering **correctly** is hard.

---

*All results are real execution outputs from a CT Lab MCP session on BTCUSDT 15m (1,724 candles). No figures are illustrative.*
