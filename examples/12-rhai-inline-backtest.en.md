# 12 — Rhai Inline: Strategies Without Files

> **Level:** Intermediate · **Prerequisites:** [Fork Doctrine](./11-fork-doutrina.en.md), [Rhai Strategy](../docs/05-backtest/03-estrategia-rhai.en.md)

In all previous examples, you wrote strategies as `estrategia_script` — a Rhai string passed directly in the `ct_backtest` JSON. But you never stopped to understand the contract: what's available inside that script? What variables can you read? What functions can you call?

This example is the practical manual for **Rhai inline** — the scripting language that executes inside the backtest. You'll see 8 classic strategies written inline, compare their results, and understand the anatomy of CT Lab's Rhai contract.

> **Rhai** is a scripting language embedded in Rust, sandboxed, no I/O, type-safe. In CT Lab, the backtest executes the script bar by bar, injecting price, position, and indicators. The script's return becomes the execution decision.

---

## The Rhai contract

Each bar, the backtest calls your script with the following variables available:

### Market variables (indexed series)

| Variable | Type | Description |
|---|---|---|
| `open[0]` | float | Current candle open price |
| `high[0]` | float | High price |
| `low[0]` | float | Low price |
| `close[0]` | float | Close price |
| `volume[0]` | float | Volume |
| `open[1]` | float | Previous candles (`[1]`, `[2]`, ...) |

### State variables

| Variable | Type | Description |
|---|---|---|
| `posicao` | float | Current position (net lots; > 0 long, < 0 short) |
| `estado` | map | Persistent state between bars (mutable object) |
| `ordens` | map | Current bar orders (filled) |
| `par` | map | Strategy parameters (`par["name"]`) |

### Indicator variables

| Variable | Type | Description |
|---|---|---|
| `ind["alias"][0]` | float | Indicator value at current candle |
| `ind["alias"][1]` | float | Value at previous candle |

### Decision functions (required return)

| Function | Description |
|---|---|
| `comprado(qty)` | Buy/hold `qty` lots |
| `vendido(qty)` | Sell/hold `qty` lots |
| `zerado()` | Flatten position |
| `decisao(target, orders)` | Decision with OCO orders (for manager) |

### Order functions (for `grupo` manager)

| Function | Description |
|---|---|
| `limite_compra(id, lot, price)` | Buy limit order |
| `limite_venda(id, lot, price)` | Sell limit order |
| `stop_compra(id, lot, price)` | Buy stop order |
| `stop_venda(id, lot, price)` | Sell stop order |
| `oco(group, order)` | OCO order (one-cancels-other) |

### Inline indicators (receitas)

Indicators can be declared as **receita** — vectorized Rhai formula computed over the price series, without pre-materializing:

```json
{
  "indicadores_receitas": {
    "rsi": { "receita": "rsi(close, 14)" },
    "sma_fast": { "receita": "sma(close, par[\"period\"])", "parametros": { "period": 20 } }
  }
}
```

> The engine materializes each receita over the series and injects as `ind["alias"]`. The receita has access to columns (`close`, `open`, `high`, `low`, `volume`) and `par` (receita parameters). The strategy has access to `ind["alias"]` and `par` (strategy parameters).

---

## Strategy 1 — SMA Crossover (20/50)

Moving average crossover: when fast MA crosses above slow MA, buy; when below, sell.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "sma_fast": { "receita": "sma(close, 20)" },
      "sma_slow": { "receita": "sma(close, 50)" }
    },
    "estrategia_script": "if ind[\"sma_fast\"][0] > ind[\"sma_slow\"][0] { comprado(1.0) } else { vendido(1.0) }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_smacross_fee"
  }
}
```

### Anatomy

```
Inputs: sma_fast = sma(close, 20)
        sma_slow = sma(close, 50)
Logic:  if sma_fast > sma_slow → comprado(1.0)
        else → vendido(1.0)
```

Always positioned — never neutral. If the fast MA is above the slow MA, long; otherwise, short. Binary trend-following.

---

## Strategy 2 — RSI Extreme (14)

RSI < 30 = oversold, buy. RSI > 70 = overbought, sell. Otherwise, flat.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_rsi_fee"
  }
}
```

Mean reversion: buy when the market has fallen too much, sell when it has risen too much. Only operates at extremes — the rest of the time stays out.

---

## Strategy 3 — Bollinger Breakout

Price breaks above upper band = buy (breakout). Price breaks below lower band = sell.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r12_bollinger",
    "estrategia_script": "if close[0] > ind[\"upper\"][0] { comprado(1.0) } else if close[0] < ind[\"lower\"][0] { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_boll_fee"
  }
}
```

> Here we use `indicadores` (pre-materialized derived series via pipeline) instead of `indicadores_receitas`, because Bollinger produces multiple columns (`upper`, `lower`, `middle`) that can't be expressed as a single receita.

---

## Strategy 4 — MACD Histogram

MACD above signal = positive momentum, buy. Below = negative momentum, sell.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r12_macd",
    "estrategia_script": "if ind[\"macd\"][0] > ind[\"signal\"][0] { comprado(1.0) } else { vendido(1.0) }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_macd_fee"
  }
}
```

Always positioned, same as SMA crossover — but using MACD histogram direction as the trigger.

---

## Strategy 5 — RSI + SMA200 (combo)

Trend filter + mean reversion: only buy if price is above SMA200 (uptrend) and RSI < 40 (pullback). Vice versa for selling.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" },
      "sma200": { "receita": "sma(close, 200)" }
    },
    "estrategia_script": "if close[0] > ind[\"sma200\"][0] && ind[\"rsi\"][0] < 40.0 { comprado(1.0) } else if close[0] < ind[\"sma200\"][0] && ind[\"rsi\"][0] > 60.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_combo_fee"
  }
}
```

Combines two concepts: trend (SMA200) and momentum (RSI). The trend filter reduces trades — only operates pullbacks in the direction of the trend.

---

## Strategy 6 — ATR Breakout

Price breaks SMA20 ± 1 ATR = volatility breakout. Buy above, sell below.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "atr": { "receita": "atr(high, low, close, 14)" },
      "sma_mid": { "receita": "sma(close, 20)" }
    },
    "estrategia_script": "if close[0] > ind[\"sma_mid\"][0] + ind[\"atr\"][0] { comprado(1.0) } else if close[0] < ind[\"sma_mid\"][0] - ind[\"atr\"][0] { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_atr_fee"
  }
}
```

The ATR band adapts to volatility: in calm markets, the band is narrow (few signals); in volatile markets, it's wide (signals only on large moves).

---

## Strategy 7 — Price Action (no indicators)

Higher high + bullish candle = buy. Lower low + bearish candle = sell. No indicators at all.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if high[0] > high[1] && close[0] > open[0] { comprado(1.0) } else if low[0] < low[1] && close[0] < open[0] { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_price_fee"
  }
}
```

The purest strategy: uses only `open`, `high`, `low`, `close` — no indicators. Shows that the Rhai contract gives direct OHLCV access without processing.

---

## Strategy 8 — Parametrized RSI

Same as Strategy 2, but with thresholds parameterized via `parametros`:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, par[\"period\"])", "parametros": { "period": 14 } }
    },
    "estrategia_script": "if ind[\"rsi\"][0] < par[\"oversold\"] { comprado(1.0) } else if ind[\"rsi\"][0] > par[\"overbought\"] { vendido(1.0) } else { zerado() }",
    "parametros": { "oversold": 25, "overbought": 75 },
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_rsi_param_fee"
  }
}
```

> Two levels of parameters: `parametros` on the receita (`period`) and `parametros` on the strategy (`oversold`, `overbought`). The receita has its own `par`, and the strategy has its own. Both are injected into the respective sandbox's `par`.

---

## Results

### With fees (0.1%)

| Strategy | Trades | PnL | Win% | PF | Sharpe | Max DD | Exposure |
|---|---|---|---|---|---|---|---|
| SMA cross (20/50) | 77 | −$14,774 | 13.3% | 0.43 | 0.021 | 147.8% | 99.9% |
| RSI extreme (30/70) | 107 | −$13,511 | 9.3% | 0.22 | 0.007 | 135.1% | 14.3% |
| Bollinger breakout | 148 | −$18,478 | 4.7% | 0.11 | 0.056 | 184.8% | 10.5% |
| MACD histogram | 101 | −$18,926 | 13.9% | 0.25 | 0.028 | 190.3% | 99.9% |
| RSI + SMA200 combo | 76 | −$9,732 | 11.8% | 0.29 | 0.029 | 97.3% | 9.7% |
| ATR breakout | 44 | −$5,597 | 18.2% | 0.43 | 0.043 | 56.0% | 4.4% |
| Price action | 822 | −$108,580 | 7.2% | 0.07 | 0.024 | 1085.8% | 82.4% |
| RSI parametrized (25/75) | 78 | −$9,893 | 11.5% | 0.06 | −0.063 | 98.9% | 11.4% |
| Buy & Hold | 0 | −$296 | — | — | 0.003 | 28.9% | 99.9% |

### Without fees

| Strategy | Trades | PnL | Win% | PF | Sharpe | Calmar | Max DD |
|---|---|---|---|---|---|---|---|
| SMA cross (20/50) | 77 | −$1,776 | 36.4% | 0.83 | 0.017 | 0.118 | 17.8% |
| RSI extreme (30/70) | 107 | +$1,369 | 77.6% | **1.82** | 0.013 | 48.49 | 12.4% |
| Bollinger breakout | 148 | +$1,627 | 72.3% | **1.55** | 0.013 | 238.77 | 12.4% |
| MACD histogram | 101 | −$5,926 | 22.8% | 0.60 | −0.026 | −1.45 | 69.1% |
| RSI + SMA200 combo | 76 | +$2,088 | 78.9% | **2.38** | 0.019 | 175.30 | 12.4% |
| ATR breakout | 44 | +$1,033 | 68.2% | **1.61** | 0.018 | 132.75 | 7.8% |
| Price action | 822 | +$2,932 | 35.9% | 1.02 | 0.009 | 4.30 | 43.3% |
| RSI parametrized (25/75) | 78 | +$114 | 71.8% | **1.03** | 0.004 | 1.23 | 21.2% |
| Buy & Hold | 0 | −$168 | — | — | 0.003 | −1.03 | 28.6% |

---

## Analysis

### Without fees: RSI + SMA200 is the best

The trend + mean reversion combo was the most profitable strategy without fees: +$2,088, profit factor 2.38, win rate 78.9%, and Calmar of 175 — max drawdown of only 12.4%. The SMA200 filter cuts counter-trend entries, keeping only quality pullbacks.

### Without fees: Bollinger and raw RSI also have edge

Bollinger breakout: PF 1.55, win 72.3%. RSI extreme: PF 1.82, win 77.6%. Both are mean reversion strategies that operate at extremes — relatively few trades, high hit rate.

### With fees: everything loses

No strategy overcomes 0.1% fee per side. Execution energy consumes all the edge. ATR breakout is the least bad (−$5,597) because it has the fewest trades — 44 entries vs 822 for price action.

### Price action is extreme overtrading

822 trades over 1712 candles (48% turnover) with $111k in fees. Without fees it has positive PnL (+$2,932) but the absurd turnover makes it impossible to trade with real costs.

> **The lesson: simple works, but cost kills.** Classic strategies (RSI, Bollinger, SMA cross) generate positive gross edge, but none overcomes 0.1% fees on BTC 15m. The path forward is reducing turnover via filters (regime from example 09) or using a manager that reduces trades (`grupo` lib from example 11).

---

## When to use receita vs derived series

| Approach | When to use | Advantage |
|---|---|---|
| `indicadores_receitas` | Indicator returns 1 column (RSI, SMA, ATR) | No pre-materialization — fast |
| `indicadores` (derived series) | Indicator returns multiple columns (Bollinger, MACD, ADX) | Named columns | 
| Pipeline + `indicadores` | Multiple composite indicators | Single URI, everything aligned |

> **Rule of thumb**: if the indicator has one column (RSI, ATR, SMA), use `indicadores_receitas`. If it has multiple (Bollinger: upper/lower/middle, MACD: macd/signal, ADX: adx/plus_di/minus_di), materialize via pipeline and pass as `indicadores`.

---

## Next steps

- **Add regime**: filter the strategies in this example with ADX < 20 (sideways) or ADX > 25 (trending) — see [Regime + Model](./09-regime-modelo.en.md).
- **Add manager**: combine strategies with the `grupo` lib for stops and trailing — see [Fork Doctrine](./11-fork-doutrina.en.md).
- **Optimize parameters**: use `parametros` to grid-search periods and thresholds — run multiple backtests with different values and compare.
- **Save strategy as file**: if preferred, save the Rhai script as a `.rhai` file and pass via `estrategia` instead of `estrategia_script` — useful for long scripts.

---

> Back to: [README](../README.md) · [Fork Doctrine](./11-fork-doutrina.en.md) · [Regime + Model](./09-regime-modelo.en.md)

_Last updated: 2026-08-12_
