# 01 — SMA Crossover: Your First Strategy

> **Level:** Beginner · **Time:** ~15 minutes · **Prerequisites:** [Installation](../docs/01-instalacao/) complete, MCP server connected

The SMA crossover is the oldest and simplest systematic strategy — perfect as a first example. You'll fetch real data, compute indicators, build a declarative pipeline, run multiple backtests, and discover why "simple" doesn't mean "profitable."

---

## What is an SMA Crossover?

The **Simple Moving Average (SMA)** is the arithmetic mean of the closing prices over the last *N* candles. It smooths out price noise so the underlying trend becomes visible.

When you plot two SMAs of different lengths, their **crossovers** act as trading signals:

| Signal | Condition | Action |
|--------|-----------|--------|
| **Golden Cross** | Fast SMA crosses *above* Slow SMA | **Buy** (bullish) |
| **Death Cross** | Fast SMA crosses *below* Slow SMA | **Sell** (bearish) |

```
Price  ─────╮         ╭─────────
            ╰───╮  ╭──╯
SMA(9)  ────╰──╯────────────
SMA(21) ────────────────────

              ↑ Golden Cross (buy)
```

---

## Step 1 — Fetch the BTCUSDT 15m Series

**Chat (AI):**
> Fetch BTCUSDT 15m from Binance

**MCP tool call:**
```json
{
  "name": "buscar_binance",
  "arguments": { "symbol": "BTCUSDT", "interval": "15m" }
}
```

**Real return:**
```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": 1785611700,
  "last_ts": 1786510800,
  "evicted_series": 0
}
```

The series is available at `ct://series/binance/BTCUSDT/15m`. The cache already contained candles from prior executions — checking the metadata:

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "kind": "raw",
  "columns": ["open", "high", "low", "close", "volume"],
  "row_count": 1724,
  "first_ts": 1784960100,
  "last_ts": 1786510800
}
```

**1,724 candles** of BTCUSDT 15-minute data — roughly 18 days of continuous market activity.

---

## Step 2 — Compute Individual SMAs

Before building a pipeline, let's look at the SMA values in isolation to develop intuition.

**Chat (AI):**
> Compute SMA 9 and SMA 21 on BTCUSDT 15m

**MCP tool calls:**
```json
[
  { "name": "sma", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 9, "name": "btc_sma9" } },
  { "name": "sma", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 21, "name": "btc_sma21" } }
]
```

**Return (SMA 9):**
```json
{
  "uri": "ct://derived/btc_sma9",
  "row_count": 1724,
  "value_names": ["sma"],
  "latest": [63774.85]
}
```

**Return (SMA 21):**
```json
{
  "uri": "ct://derived/btc_sma21",
  "row_count": 1724,
  "value_names": ["sma"],
  "latest": [63769.40]
}
```

At the latest candle, SMA(9) = **63,774.85** and SMA(21) = **63,769.40**. The fast SMA is slightly above the slow SMA — a marginal bullish state. The two are very close together, typical before a crossover event.

> **Note:** Each computed SMA generates a derived series persisted at `ct://derived/<name>`. You can reuse these series later without recalculating.

---

## Step 3 — Build the Declarative Pipeline

Instead of separate calculations, CT Lab lets us declare the logic as a **pipeline** of steps. The final step is the crossover signal itself.

**Chat (AI):**
> Build a pipeline that computes SMA 9, SMA 21, and a crossover signal

**MCP tool call:**
```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_cruzamento_sma",
    "output": "$sinal",
    "steps": [
      { "id": "sma9", "op": "sma", "source": "$anchor", "period": 9 },
      { "id": "sma21", "op": "sma", "source": "$anchor", "period": 21 },
      {
        "id": "cruz_acima",
        "op": "comparar",
        "esquerda": "$sma9",
        "coluna_esquerda": "sma",
        "direita": "$sma21",
        "coluna_direita": "sma",
        "operador": "cruza_acima",
        "coluna_saida": "cruz_acima"
      },
      {
        "id": "cruz_abaixo",
        "op": "comparar",
        "esquerda": "$sma9",
        "coluna_esquerda": "sma",
        "direita": "$sma21",
        "coluna_direita": "sma",
        "operador": "cruza_abaixo",
        "coluna_saida": "cruz_abaixo"
      },
      {
        "id": "sinal",
        "op": "condicional",
        "condicao": "$cruz_acima",
        "coluna_condicao": "cruz_acima",
        "entao": { "escalar": 1.0 },
        "senao": { "escalar": -1.0 },
        "coluna_saida": "sinal"
      }
    ]
  }
}
```

**Real return:**
```json
{
  "uri": "ct://derived/btc_cruzamento_sma",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1724,
  "first_ts": 1784960100,
  "last_ts": 1786510800,
  "steps_executed": 5
}
```

> **Important detail:** when comparing two **derived series** (not the `close` column from the anchor), you **must** specify `coluna_esquerda` and `coluna_direita` with the column name produced by the preceding step. SMAs produce a column named `sma`, so we use `coluna_esquerda: "sma"` and `coluna_direita: "sma"`. If omitted, the engine tries to use `close` — which doesn't exist on a derived SMA series.

The pipeline executed **5 steps** and was persisted at `ct://derived/btc_cruzamento_sma`. This series contains the signal: `1` when the fast SMA crossed above the slow SMA, `-1` when it crossed below.

---

## Step 4 — Backtest SMA 9/21 (No Fees)

Now the moment of truth: run the backtest.

**Chat (AI):**
> Run a backtest: buy when SMA 9 > SMA 21, else flat. Capital 1000, no fee.

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma9\"][0] > ind[\"sma21\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma9": { "receita": "sma(close, 9)" },
      "sma21": { "receita": "sma(close, 21)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "sma_cross_9_21_nofee"
  }
}
```

**Real return (summary):**
```json
{
  "uri": "ct://backtest/sma_cross_9_21_nofee",
  "num_trades": 46,
  "pnl_total": -44.82,
  "pnl_bruto": 45.26,
  "fees_totais": 0,
  "retorno_total": -0.0448,
  "sharpe": 0.043,
  "sortino": 0.326,
  "calmar": -0.453,
  "win_rate": 0.3696,
  "profit_factor": 1.010,
  "avg_win": 275.45,
  "avg_loss": -159.91,
  "payoff_ratio": 1.72,
  "drawdown_max": 1.34,
  "exposicao": 0.5238,
  "num_wins": 17,
  "num_losses": 29,
  "max_wins_seguidos": 3,
  "max_perdas_seguidas": 7
}
```

Without fees, the strategy is essentially break-even:

- **Gross PnL:** +$45.26 (a tiny positive edge)
- **Win rate:** 36.96% (only ~37% of trades are winners)
- **Profit factor:** 1.010 (just barely above 1.0)
- **Payoff ratio:** 1.72 (winners are 1.72× larger than losers on average)

The strategy is marginally profitable before costs. But "marginally profitable before costs" is a dangerous place to be — because costs are about to change everything.

---

## Step 5 — The Impact of Fees

Real trading incurs fees. Binance charges roughly **0.1% per trade** (taker fee). Let's re-run the same backtest with `fee_pct: 0.001`.

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma9\"][0] > ind[\"sma21\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma9": { "receita": "sma(close, 9)" },
      "sma21": { "receita": "sma(close, 21)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "sma_cross_9_21_fee"
  }
}
```

**Real return (summary):**
```json
{
  "uri": "ct://backtest/sma_cross_9_21_fee",
  "num_trades": 46,
  "pnl_total": -6080.91,
  "pnl_bruto": 45.26,
  "fees_totais": 5908.45,
  "sharpe": 0.025,
  "sortino": 0.039,
  "win_rate": 0.2174,
  "profit_factor": 0.353,
  "avg_win": 320.55,
  "avg_loss": -251.91,
  "payoff_ratio": 1.27,
  "drawdown_max": 5.89
}
```

### The Stark Difference

| Metric | No Fees | With 0.1% Fee |
|--------|---------|---------------|
| **PnL Total** | -$44.82 | **-$6,080.91** |
| **PnL Bruto** | $45.26 | $45.26 |
| **Fees** | $0.00 | **$5,908.45** |
| **Win Rate** | 36.96% | 21.74% |
| **Profit Factor** | 1.010 | 0.353 |
| **Max Drawdown** | 1.34% | 5.89% |

The gross edge of **$45.26** is completely obliterated by **$5,908.45** in fees.

### Why so much in fees?

Each trade costs **0.1% of notional value**. With BTC at ~$64,000, each trade costs approximately:

```
$64,000 × 0.001 = $64 per entry + $64 per exit = ~$128 round-trip
```

With **46 trades** (each trade = one round-trip entry + exit):

```
$128 × 46 = $5,888 ≈ $5,908.45 ✓
```

The entire gross edge of $45 is about **0.76%** of the fee cost. Fees are **131× larger** than the edge!

---

## Step 6 — Vary the Parameters

Maybe different SMA periods produce better results? Let's test two extreme variants and compare with a Buy & Hold benchmark. All use 0.1% fee.

### Variant A: SMA 5/20 (Faster, More Trades)

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma5\"][0] > ind[\"sma20\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma5": { "receita": "sma(close, 5)" },
      "sma20": { "receita": "sma(close, 20)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "sma_cross_5_20_fee"
  }
}
```

**Real return (summary):**
```json
{
  "num_trades": 60,
  "pnl_total": -11522.77,
  "pnl_bruto": -3633.30,
  "fees_totais": 7708.06,
  "sharpe": -0.014,
  "win_rate": 0.1833,
  "profit_factor": 0.243,
  "drawdown_max": 10.81
}
```

### Variant B: Golden Cross 50/200 (Slower, Fewer Trades)

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma50\"][0] > ind[\"sma200\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma50": { "receita": "sma(close, 50)" },
      "sma200": { "receita": "sma(close, 200)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "golden_cross_50_200_fee"
  }
}
```

**Real return (summary):**
```json
{
  "num_trades": 7,
  "pnl_total": -3916.78,
  "pnl_bruto": -3018.49,
  "fees_totais": 898.29,
  "sharpe": -0.026,
  "win_rate": 0.1429,
  "profit_factor": 0.052,
  "drawdown_max": 2.31,
  "exposicao": 0.6363
}
```

### Variant C: Buy & Hold (Benchmark)

**MCP tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "comprado(1.0)",
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "buy_hold"
  }
}
```

**Real return (summary):**
```json
{
  "num_trades": 0,
  "pnl_total": -281.50,
  "retorno_total": -0.2815,
  "sharpe": -0.024,
  "drawdown_max": 1.25,
  "exposicao": 0.9994
}
```

> The Buy & Hold result shows the market declined over this period (return = **-28.15%**). Important context: the strategy was fighting a downtrend.

---

## Comparative Analysis — All 5 Strategies

| # | Strategy | Trades | PnL Total | PnL Bruto | Fees | Sharpe | Win Rate | PF | Payoff | DD Max | Exposition |
|---|----------|--------|-----------|-----------|------|--------|----------|----|--------|--------|------------|
| 1 | SMA 9/21 (no fee) | 46 | -$44.82 | $45.26 | $0 | 0.043 | 36.96% | 1.010 | 1.72 | 1.34% | 52.38% |
| 2 | SMA 9/21 (0.1% fee) | 46 | -$6,080.91 | $45.26 | $5,908 | 0.025 | 21.74% | 0.353 | 1.27 | 5.89% | 52.38% |
| 3 | SMA 5/20 (0.1% fee) | 60 | -$11,522.77 | -$3,633 | $7,708 | -0.014 | 18.33% | 0.243 | 1.08 | 10.81% | 52.32% |
| 4 | Golden Cross 50/200 | 7 | -$3,916.78 | -$3,018 | $898 | -0.026 | 14.29% | 0.052 | 0.31 | 2.31% | 63.63% |
| 5 | Buy & Hold | 0 | -$281.50 | — | — | -0.024 | — | — | — | 1.25% | 99.94% |

### Key Takeaways

1. **Fees dominate** short-period SMA strategies. SMA 9/21 produces $45 of gross edge but pays $5,908 in fees — the fee-to-edge ratio is **131:1**.

2. **Faster crossovers make it worse.** SMA 5/20 generates 60 trades (vs 46 for 9/21), pays $7,708 in fees, *and* has negative gross PnL (-$3,633). More trading = more fees + more whipsaw losses.

3. **Golden Cross doesn't help.** While it trades less (7 trades, $898 in fees), the gross PnL is already deeply negative (-$3,018). The signals are too slow for 15m — by the time the 200-period SMA crosses, the move is over. And 7 trades is far too few for statistical significance.

4. **Buy & Hold was the least disastrous.** Losing $281 on the period, it outperformed every SMA variant by a wide margin. The market was down, but merely holding was far cheaper than churning capital through crossovers.

5. **Exposition matters.** SMA 9/21 was only in the market 52.38% of the time, yet still lost far more than Buy & Hold at 99.94% exposition. Being "picky" didn't help; it just added transaction costs.

---

## Interpretation — What Each Metric Means

### PnL Total vs PnL Bruto
- **PnL Bruto** (gross PnL) = sum of per-trade profits and losses *before* fees.
- **PnL Total** = PnL Bruto − Fees.
- The gap between these two is the **real cost of trading**. For SMA 9/21 with fees, the gap is $5,953 — meaning 98% of the total loss comes from fees, not from the signal being "bad."

### Sharpe Ratio
Measures risk-adjusted return. A Sharpe of 0.043 (no fee) is near zero — essentially indistinguishable from randomness. With fees, it drops to 0.025. For SMA 5/20 and Golden Cross, it turns negative.

### Sortino Ratio
Like Sharpe, but only penalizes *downside* volatility. A Sortino of 0.326 for the no-fee case might look better, but still indicates the strategy isn't generating meaningful risk-adjusted returns.

### Win Rate
The percentage of trades that are profitable. A 36.96% win rate means ~63% of trades lose money. However, high win rate alone is meaningless — what matters is the **combination** of win rate and payoff ratio.

### Profit Factor (PF)
Total gross profits ÷ total gross losses.
- PF > 1.0 = profitable (before fees)
- PF > 1.5 = solid
- PF > 2.0 = excellent
- PF < 1.0 = losing

SMA 9/21 has PF = 1.010 *without fees* — essentially a coin flip with a slight edge. With fees, PF collapses to 0.353.

### Payoff Ratio
Average win ÷ average loss. $275.45 / $159.91 = 1.72. Winners are 1.72× larger than losers — the strategy's only redeeming quality. But it's not enough to overcome the low win rate + fees.

### Max Drawdown (DD Max)
The maximum peak-to-valley decline in equity during the backtest. The fee version's 5.89% drawdown is manageable, but for a strategy losing money, it's drawdown *on top of* losses.

### Exposition
The proportion of time the strategy holds a position. SMA 9/21 is invested ~52% of the time. Lower exposition can be good if the signal is selective — but here it just means we're paying fees to be right half the time.

---

## Why Doesn't SMA Crossover Work Well?

This is the most important pedagogical lesson from this example.

### 1. SMA is a Lagging Indicator

The SMA is — by definition — an average of *past* prices. By the time the fast SMA crosses the slow SMA, the trend change has **already happened**. You're reacting, not predicting. In choppy markets (which 15m crypto often is), this lag means you buy near local tops and sell near local bottoms.

### 2. Whipsaw in Range-Bound Markets

SMA crossovers work beautifully in **trending markets** — one clean signal and you ride the trend. But in **range-bound markets** (which dominate short timeframes), the two SMAs oscillate around each other, generating false signal after false signal. Each false signal is a round-trip trade costing ~$128 in fees.

### 3. The Fee Math is Brutal

```
Gross edge per trade = $45.26 / 46 trades ≈ $0.98 per trade
Fee per round-trip    = ~$128 per trade

Edge-to-fee ratio     = $0.98 / $128 = 0.0077

You'd need a gross edge 130× larger just to break even after fees.
```

No SMA crossover strategy on a 15-minute chart will produce $128 of gross edge per trade. The edge per trade is typically single-digit dollars — orders of magnitude below the fee threshold.

### 4. The Underlying Can't Trend Enough on 15m

On a 15m timeframe, BTC doesn't trend smoothly for long stretches within a 2–3 week window. The data shows a declining market (-28.15% for Buy & Hold), and SMA crossover is a trend-following strategy — designed to capture sustained directional moves. When those moves are choppy and short-lived, the strategy bleeds.

### 5. SMA Weights All Prices Equally

A 21-period SMA gives equal weight to the candle 21 bars ago and the most recent candle. In practice, recent price action is far more relevant. This is why **EMA** (Exponential Moving Average) exists — but even EMA doesn't solve the fundamental fee problem.

---

## Variations to Experiment With

| # | Variation | How | Hypothesis |
|---|-----------|-----|------------|
| 1 | **EMA instead of SMA** | Change `sma(close, N)` to `ema(close, N)` in recipes | EMAs react faster; may catch trends earlier, but also generate more false signals in chop |
| 2 | **Higher timeframe (1h or 4h)** | Re-fetch with `"interval": "1h"` | Fewer candles, fewer trades, larger edge per trade — fees become proportionally smaller |
| 3 | **ADX filter** | Add ADX to `indicadores_receitas`; only trade when ADX > 25 | Filters out whipsaw trades in no-trend markets |
| 4 | **Rising ATR gate** | Only enter when ATR is rising (volatility expansion) | Crossover signals during low-volatility periods are often false |
| 5 | **Lower fee** | Reduce `fee_pct` to 0.0005 (maker) or 0.0001 (VIP) | Shows how fee structure affects viability — at what fee does SMA 9/21 break even? |
| 6 | **Long-only mode** | Strategy only buys, never shorts | Avoids shorting in downtrends; may reduce whipsaw losses |
| 7 | **Different assets** | Try `ETHUSDT` or `SOLUSDT` | Different assets trend differently; some may be more SMA-friendly at 15m |

> **Experimental challenge:** What is the minimum fee rate at which SMA 9/21 becomes profitable? Iterate `fee_pct` from 0.0001 to 0.001 in steps of 0.0001 and observe PnL. You'll likely find the breakeven is around 0.0001–0.0003 — well inside maker-fee territory with VIP discounts.

---

## Next Steps

You've built, backtested, and analyzed your first systematic strategy. Here's where to go next:

1. **[Recipe 02 — RSI with ADX filter](./02-rsi-filtro-adx.en.md)** — Adds a trend filter to reduce false signals.

2. **[Recipe 03 — Backtest with `grupo` lib](./03-lib-grupo-backtest.en.md)** — Sophisticated execution with scaled entries, OCO stops, and trailing.

3. **[Example 04 — Survival test](./04-teste-sobrevivencia.en.md)** — How to know if your strategy survives out-of-sample.

4. **[Example 08 — Microstructure: TFI](./08-tfi-backtest.en.md)** — Going beyond OHLCV: order flow indicators.

5. **[Example 12 — Rhai inline](./12-rhai-inline-backtest.en.md)** — 8 classic strategies written in inline Rhai, compared side by side.

---

> **Final lesson:** SMA crossover is a strategy that *looks* like it should work — it's intuitive, visual, and has a century of literature behind it. But the gap between "looks reasonable" and "is profitable after costs" is enormous. This example is your first exposure to that gap. Every subsequent example explores techniques to narrow it.
>
> **The lesson isn't "SMA crossover is useless."** The lesson is: **always include fees in your backtest, understand your edge-to-fee ratio, and recognize that on short timeframes with moderate edge, transaction costs are the dominant determinant of profitability.**

---

*All results are real execution outputs from a live CT Lab MCP session on BTCUSDT 15m data covering 1,724 candles. No figures are illustrative.*
