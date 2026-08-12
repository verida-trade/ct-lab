# 08 — Microstructure: TFI, OBI, BFI & Backtest

> **Level:** Advanced · **Premium** · **Prerequisites:** [GBDT Model](./06-modelo-gbdt.en.md), [MLP Model](./07-modelo-lstm.en.md)

In examples 06 and 07 you trained ML models to predict the next candle's direction — features built from classic indicators (RSI, MACD, ADX) over 15-minute OHLCV. But all OHLCV is an **aggregation**: it compresses thousands of trades and millions of book updates into 4 prices and 1 volume. The information lost in that compression is **microstructure** — who is buying, who is selling, and how urgently.

In this example you'll work with **1-second microstructure data** (Premium): the taker flow (trades) and the order book state (bid/ask). You'll compute 4 microstructure indicators and analyze them side-by-side, at the same timestamp, to understand how buying pressure manifests at each layer of the market.

> **Microstructure** is the study of price formation at the most granular level — who initiates liquidity (maker), who takes it (taker), and how the imbalance between the two reveals the direction of informed flow.

---

## The problem

You want to measure **buying vs selling pressure** in real time. There are several ways to measure it:

1. **TFI (Trade Flow Imbalance)** — from executed trades: `qty_delta / qty`. If takers are buying more than selling, TFI > 0.
2. **OBI (Order Book Imbalance)** — from top-of-book: `(bid_qty − ask_qty) / (bid_qty + ask_qty)`. If there's more buy liquidity at best price, OBI > 0.
3. **BFI (Book Flow Imbalance)** — from book changes: `(bid_qty_delta − ask_qty_delta) / (|bid_qty_delta| + |ask_qty_delta|)`. If the book is being filled more on the buy side, BFI > 0.
4. **DBI (Depth Book Imbalance)** — from deep layers: `(depth_bid − depth_ask) / sum`. Same as OBI but looking ±0.1% or ±1% from mid.

The difference between them is the **observation point**:

| Indicator | Source | Question | Granularity |
|---|---|---|---|
| TFI | Trades | Who is **taking** liquidity? | 1s |
| OBI | Book top | Where is the **liquidity**? | 1s |
| BFI | Book deltas | How did the book **change**? | 1s |
| DBI ±0.1% | Deep book | Where is the **wall**? | 1s |

> TFI measures **action** (market orders executed). OBI/BFI/DBI measure **intent** (limit orders in the book). When TFI and OBI agree, the pressure is unambiguous. When they diverge, something is happening — perhaps absorption.

---

## Step 1 — Start live collectors (Premium)

First, enable the trades and book collectors. They operate at 1-second resolution and accumulate data continuously via WebSocket.

### 1.1 — Trades collector

```json
{
  "name": "coletar_trades",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "backfill_dias": 1 }
}
```

### 1.2 — Book collector

```json
{
  "name": "coletar_book",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT" }
}
```

### 1.3 — Check collector status

```json
{ "name": "coletas_ativas", "arguments": {} }
```

Both return a `collector_id` — the raw series URI:

| Series | URI | Granularity |
|---|---|---|
| Trades | `ct://series/binance/BTCUSDT/trades_1s` | 1s |
| Book | `ct://series/binance/BTCUSDT/book_1s` | 1s |

> `backfill_dias: 1` on trades pulls historical data from `data.binance.vision` (daily dump) before starting the live WebSocket. Book has no backfill — Binance Spot doesn't offer bulk L2.

---

## Step 2 — Query microstructure data

The `trades_1s` and `book_1s` series are massive (hundreds of thousands of rows at 1s). To inspect without overload, use `consultar_trades` and `consultar_book` — both accept `t0`, `t1`, and `agregacao`:

### 2.1 — Query trades in 1m buckets

```json
{
  "name": "consultar_trades",
  "arguments": {
    "provider": "binance", "symbol": "BTCUSDT",
    "t0": 1786504560, "t1": 1786505220,
    "agregacao": "1m"
  }
}
```

Result (11 one-minute buckets):

| ts | n_trades | qty | qty_delta | TFI | price_open | price_close |
|---|---|---|---|---|---|---|
| 1786504560 | 160 | 0.96 | +0.004 | +0.004 | 63722.00 | 63715.19 |
| 1786504620 | 178 | 5.18 | +3.17 | +0.613 | 63715.20 | 63715.63 |
| 1786504680 | 337 | 7.47 | +0.63 | +0.084 | 63715.63 | 63724.63 |
| 1786504800 | 73 | 1.58 | −0.59 | −0.373 | 63724.63 | 63722.73 |
| 1786504860 | 155 | 3.59 | +2.98 | +0.831 | 63722.72 | 63734.24 |
| 1786504920 | 131 | 4.80 | +4.46 | +0.930 | 63734.23 | 63745.92 |
| 1786504980 | 135 | 5.89 | −0.31 | −0.052 | 63745.92 | 63745.92 |
| 1786505040 | 145 | 1.87 | +1.43 | +0.766 | 63745.92 | 63757.16 |
| 1786505100 | 125 | 1.05 | +0.37 | +0.358 | 63757.16 | 63765.05 |
| 1786505160 | 185 | 4.80 | −3.14 | −0.655 | 63765.04 | 63776.38 |
| 1786505220 | 300 | 3.72 | −3.05 | −0.822 | 63776.37 | 63740.81 |

> **TFI = qty_delta / qty.** In bucket `4920`, qty_delta=+4.46 with qty=4.80 → TFI=+0.93: 93% of volume was net buying. Price rose from 63734 → 63746. TFI worked as a directional signal.

### 2.2 — Query book in 1s buckets (with top-of-book columns)

```json
{
  "name": "consultar_book",
  "arguments": {
    "provider": "binance", "symbol": "BTCUSDT",
    "t0": 1786505259, "t1": 1786505319,
    "agregacao": "1s",
    "colunas": ["bid_qty_top", "ask_qty_top", "bid_qty_delta", "ask_qty_delta",
                "depth_0_1pct_bid", "depth_0_1pct_ask", "depth_1pct_bid", "depth_1pct_ask",
                "spread_bps", "mid_close", "bid_price", "ask_price"]
  }
}
```

Result (first 10 one-second buckets):

| ts | bid_top | ask_top | OBI | bid_delta | ask_delta | BFI | DBI±0.1% | DBI±1% | mid |
|---|---|---|---|---|---|---|---|---|---|
| 1786505301 | 5.22 | 1.56 | +0.54 | +0.08 | −0.15 | +1.00 | −0.034 | +0.088 | 63740.81 |
| 1786505302 | 5.22 | 0.77 | +0.74 | −0.00 | −1.82 | +0.99 | −0.022 | +0.093 | 63740.81 |
| 1786505303 | 5.24 | 0.25 | +0.91 | −0.00 | −4.03 | +1.00 | +0.096 | +0.126 | 63740.81 |
| 1786505304 | 8.16 | 0.25 | +0.94 | −3.41 | −13.69 | +0.60 | +0.064 | +0.115 | 63747.12 |
| 1786505305 | 8.16 | 0.25 | +0.94 | +2.16 | +2.59 | −0.09 | +0.078 | +0.114 | 63747.12 |
| 1786505306 | 8.22 | 0.25 | +0.94 | −0.08 | +2.32 | −1.00 | +0.070 | +0.111 | 63747.12 |
| 1786505307 | 9.57 | 0.34 | +0.93 | +0.77 | −3.07 | +1.00 | +0.062 | +0.108 | 63751.55 |
| 1786505308 | 9.57 | 0.34 | +0.93 | +2.25 | −0.93 | +1.00 | +0.066 | +0.107 | 63751.55 |
| 1786505309 | 8.39 | 0.20 | +0.95 | +1.87 | −3.27 | +1.00 | +0.093 | +0.117 | 63751.99 |
| 1786505310 | 8.38 | 0.20 | +0.95 | +2.40 | +3.13 | −0.13 | +0.090 | +0.115 | 63751.99 |

> **Reading:** OBI near +0.95 means there's ~10x more buy liquidity than sell at top-of-book. Mid rose from 63740 → 63752 in 10 seconds. The book is "saying" makers want to buy — and price responded.

---

## Step 3 — Cross TFI × OBI × BFI × DBI at the same timestamp

The real value of microstructure is in **crossing** the indicators. TFI comes from trades; OBI/BFI/DBI from the book. When you align them in the same second, you see the maker-taker dynamic:

```json
{
  "name": "consultar_trades",
  "arguments": {
    "provider": "binance", "symbol": "BTCUSDT",
    "t0": 1786505300, "t1": 1786505360,
    "agregacao": "1s"
  }
}
```

Crossing TFI (trades) with OBI/BFI/DBI (book) by timestamp:

| ts | TFI | OBI | BFI | DBI±0.1% | DBI±1% | mid |
|---|---|---|---|---|---|---|
| 1786505301 | +1.00 | +0.539 | +1.00 | −0.034 | +0.088 | 63740.81 |
| 1786505303 | +0.99 | +0.907 | +1.00 | +0.096 | +0.126 | 63740.81 |
| 1786505304 | +1.00 | +0.940 | +0.60 | +0.064 | +0.115 | 63747.12 |
| 1786505307 | +0.98 | +0.931 | +1.00 | +0.062 | +0.108 | 63751.55 |
| 1786505309 | +0.79 | +0.953 | +1.00 | +0.093 | +0.117 | 63751.99 |
| 1786505315 | −0.39 | +0.886 | −1.00 | +0.050 | +0.097 | 63755.58 |
| 1786505316 | −0.82 | +0.844 | −1.00 | +0.040 | +0.094 | 63755.58 |
| 1786505319 | −1.00 | +0.953 | +0.94 | +0.045 | +0.095 | 63755.58 |
| 1786505352 | −1.00 | −0.324 | −0.93 | −0.087 | +0.084 | 63756.58 |
| 1786505353 | −1.00 | +0.269 | +0.04 | −0.040 | +0.082 | 63756.58 |

> **Divergence at 1786505315–316:** TFI turned negative (taker selling) but OBI remains +0.88 (book still has buy-side liquidity). This is **absorption** — someone is selling into a buy wall. When the wall gives way (ts 5352: OBI turns −0.32), price drops.

---

## Step 4 — Compute indicators as derived series

The `tfi`, `obi`, `bfi`, `dbi01`, `dbi1` tools compute each indicator over the raw series and persist as derived series. `ct_tfi`, `ct_obi`, `ct_bfi` produce the VWMA windowed versions (weighted by qty/activity):

### 4.1 — TFI instant and windowed

```json
{
  "name": "tfi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/trades_1s" }
}
```

```json
{
  "name": "ct_tfi",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/trades_1s",
    "period": 15,
    "name": "btc_tfi_15s"
  }
}
```

> `tfi` computes `qty_delta / qty` per 1s bucket. `ct_tfi` does VWMA windowed: `Σ(tfi_i · qty_i) / Σ(qty_i)` over a 15-second window — high-volume buckets weigh more.

### 4.2 — OBI, BFI, DBI from book

```json
{ "name": "obi",  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

```json
{ "name": "bfi",  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

```json
{ "name": "dbi01", "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

```json
{ "name": "dbi1",  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

### 4.3 — Windowed versions (VWMA)

```json
{
  "name": "ct_obi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s", "period": 15, "name": "btc_obi_15s" }
}
```

```json
{
  "name": "ct_bfi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s", "period": 15, "name": "btc_bfi_15s" }
}
```

> Windowed versions smooth 1s noise. In ct_obi, 15s of VWMA weighted by `(bid_qty_top + ask_qty_top)` gives more weight to high-liquidity buckets — the book's "consensus".

---

## Step 5 — Backtest with BOP (OHLCV proxy for flow)

Microstructure series (trades_1s, book_1s) are at 1-second resolution. `ct_backtest` operates on OHLCV at larger timeframes (e.g. 15m). You can't join 1s and 15m via `compor_serie` — the timeframes are incompatible.

The solution is **BOP (Balance of Power)** — the OHLCV analog of TFI. BOP measures buying pressure within each candle: `(close − open) / (high − low)`. It ranges [-1, +1] and is computable directly from the price series, without trades.

### 5.1 — Materialize ct_bop (windowed)

```json
{
  "name": "ct_bop",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "period": 14,
    "name": "btc_ct_bop_14"
  }
}
```

> `ct_bop` computes per-candle BOP then does a 14-period VWMA weighted by volume. Result: `ct://derived/btc_ct_bop_14` with 1712 rows (same timeframe as the 15m series).

### 5.2 — Backtest with threshold 0.2

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_ct_bop_14",
    "estrategia_script": "if ind[\"ct_bop\"][0] > 0.2 { comprado(1.0) } else if ind[\"ct_bop\"][0] < -0.2 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "tfi_bop_fee"
  }
}
```

### 5.3 — Results

| Metric | BOP with fee 0.1% | BOP no fee | Buy & Hold |
|---|---|---|---|
| **PnL** | −$16,178 | +$398 | −$296 |
| **Gross PnL** | +$398 | +$398 | 0 |
| **Fees** | $16,576 | 0 | 0 |
| **Trades** | 129 | 129 | 0 |
| **Win rate** | 11.6% | 35.7% | — |
| **Profit factor** | 0.20 | 1.06 | 0 |
| **Expectancy** | −$125/trade | +$3.1/trade | 0 |
| **Payoff** | 1.50 | 1.91 | — |
| **Exposure** | 27.5% | 27.5% | 99.9% |
| **Sharpe** | −0.014 | 0.007 | 0.003 |
| **Sortino** | −0.016 | 0.012 | 0.004 |
| **Calmar** | −0.62 | 6.94 | −1.59 |
| **Max DD** | 161.3% | 17.7% | 28.9% |
| **Annual return** | −100% | +122.6% | −45.9% |

### Analysis

**Without fees, BOP has positive edge.** PnL of +$398, profit factor 1.06, win rate 35.7%, and Calmar of 6.94 — max drawdown is only 17.7%. Expectancy per trade is +$3.1 — each trade adds 0.03% to capital on average.

**With 0.1% fees, the edge is destroyed.** Fees eat $16,576 — 42x the gross PnL. Win rate plunges from 35.7% to 11.6% not because the number of wins changed (same 129 trades), but because fees turn marginal winners into losers. Expectancy becomes −$125/trade.

> **The problem is turnover.** 129 trades over 1712 candles (7.5% of candles) at 27.5% exposure is high for BOP on 15m. Each entry+exit costs 0.2% of notional — 129 trades × 2 sides × 0.1% = $16,576 in fees on $10,000 capital.

### 5.4 — Higher threshold reduces trades but doesn't fix it

| Threshold | Trades | PnL with fee | Gross PnL | Fees | Win rate |
|---|---|---|---|---|---|
| 0.2 | 129 | −$16,178 | +$398 | $16,576 | 11.6% |
| 0.3 | 50 | −$5,842 | +$579 | $6,421 | 10.0% |

Threshold 0.3 cuts trades from 129 to 50 and improves gross PnL (+$579 vs +$398), but fees still consume everything. There isn't enough edge to overcome execution costs.

---

## Step 6 — Why BOP doesn't beat fees (and TFI would be worse)

BOP is a **proxy** for TFI built from OHLCV. Real TFI has two additional problems:

1. **1s signal is far more frequent** — every second of trades generates a signal. That would be thousands of trades per day in a backtest, with proportional fees.
2. **TFI is extreme by design** — in 1s buckets with few trades, TFI is often ±1.0 (all trades on one side). This generates near-continuous signals with very high turnover.

The structural issue is: **microstructure has far more signals than OHLCV, but each signal carries less predictive edge.** The TFI+OBI set converges to the same direction as BOP — but with 100x more trades.

> This is why microstructure is used for **execution** (TWAP/VWAP, entry/exit timing, absorption detection) rather than **discretionary signals** on 15m. TFI brilliantly detects buying pressure right now — but doesn't predict whether price will rise over the next 15 minutes.

---

## When microstructure helps

| Scenario | How to use |
|---|---|
| **Entry timing** | TFI > 0.8 confirms buying pressure — enter long with confidence |
| **Absorption detection** | Negative TFI + positive OBI = sellers being absorbed — possible reversal |
| **Breakout confirmation** | Price breaks level + BFI > 0.5 = book is betting on the direction |
| **False signal filter** | BOP says buy but TFI < 0 = divergence — don't enter |
| **Slippage estimation** | Spread + depth → estimate execution cost before entering |

> **Microstructure doesn't replace strategy — it informs it.** Use TFI/OBI/BFI as a **filter** on top of a model's signals (like the GBDT from example 06), not as a standalone signal.

---

## Next steps

- **Combine BOP + GBDT**: add BOP as an extra feature in the example 06 pipeline — the model learns to interpret buying pressure alongside RSI/MACD/ADX.
- **Regime detection**: use TFI + BOP to detect whether the market is in directional or sideways regime — see [Regime + Model](./09-regime-modelo.en.md).
- **Fork the doctrine**: use microstructure as a filter on the adaptive manager — enter only when TFI and OBI converge — see [Fork Doctrine](./11-fork-doutrina.en.md).
- **Live timing**: use `consultar_trades` and `consultar_book` in real time for execution timing within the adaptive manager.

---

> Back to: [README](../README.md) · [GBDT Model](./06-modelo-gbdt.en.md) · [MLP Model](./07-modelo-lstm.en.md) · [Regime + Model](./09-regime-modelo.en.md)

_Last updated: 2026-08-12_
