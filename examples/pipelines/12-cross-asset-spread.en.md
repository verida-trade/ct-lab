# Recipe 12 — Cross-Asset Spread (BTC vs ETH)

> **Level:** Intermediate
> **Prerequisites:** Recipes 01–04 (series, backtest, indicators), familiarity with `compor_serie`

---

## The Concept

- **Cross-asset relative value**: instead of looking at BTC's absolute price, we use the *ratio* between two correlated assets (BTC/ETH) as a signal.
- The BTC/ETH ratio measures how much BTC is worth in ETH terms. When the ratio rises, BTC is relatively stronger; when it falls, ETH is outperforming.
- **Momentum hypothesis**: if the ratio is rising, BTC should continue outperforming → go long BTC.
- **Reversal (mean-reversion) hypothesis**: if the ratio has risen too far, it tends to revert → go long BTC when the ratio falls (betting BTC will recover).
- This recipe shows how to fetch two series, compose them into a derived series, and use the values as an indicator in the backtest.

---

## Step 1 — Fetch both series

BTC was already cached (`ct://series/binance/BTCUSDT/15m`, 1724 candles). For ETH, we use `buscar_binance_historico` aligning the time range with BTC via `since`/`until` (unix seconds as strings):

```json
{
  "name": "buscar_binance_historico",
  "arguments": {
    "symbol": "ETHUSDT",
    "interval": "15m",
    "since": "1784960100",
    "until": "1786510800"
  }
}
```

**Result:**

| Field | Value |
|---|---|
| uri | `ct://series/binance/ETHUSDT/15m` |
| row_count | 1724 |
| first_ts | 1784960100 |
| last_ts | 1786510800 |
| chunks | 2 |
| concurrency | 8 |

> Both series have 1724 candles with the same time range — essential for `compor_serie` to work correctly.

---

## Step 2 — Compose derived series (BTC + ETH)

We use `compor_serie` with BTC as the **anchor** and ETH as an additional column:

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "btc_eth_v3",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth" }
    ]
  }
}
```

**Result:**

| Field | Value |
|---|---|
| uri | `ct://derived/btc_eth_v3` |
| row_count | 1724 |
| first_ts | 1784960100 |
| last_ts | 1786510800 |

> The derived series has `btc` and `eth` columns aligned by timestamp. The backtest will use `indicadores: "ct://derived/btc_eth_v3"` and the strategy reads `ind["btc"]` and `ind["eth"]`.

---

## Step 3 — Backtest momentum (ratio rising → long BTC)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "let ratio = ind[\"btc\"][0] / ind[\"eth\"][0]; let ratio_prev = ind[\"btc\"][5] / ind[\"eth\"][5]; if ratio > ratio_prev { comprado(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_eth_v3",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "cross_mom_fee"
  }
}
```

**Logic:** `ratio` = BTC/ETH now; `ratio_prev` = BTC/ETH 5 candles ago. If the ratio rose, go long BTC.

---

## Step 4 — Backtest reversal (ratio falling → long BTC)

Same structure, reversing the condition:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "let ratio = ind[\"btc\"][0] / ind[\"eth\"][0]; let ratio_prev = ind[\"btc\"][5] / ind[\"eth\"][5]; if ratio < ratio_prev { comprado(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_eth_v3",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "cross_rev_fee"
  }
}
```

**Logic:** if the ratio fell (BTC weakened relatively), go long BTC betting on mean-reversion.

---

## Real Results

| Strategy | Trades | Net P&L | Gross | Fees | Sharpe | Win% | PF | DD% |
|---|---|---|---|---|---|---|---|---|
| Momentum (fee) | 198 | −$29,204.00 | −$3,765.62 | $25,438.38 | 0.043 | 14.6% | 0.105 | 29.2% |
| Momentum (nofee) | 198 | −$3,765.62 | −$3,765.62 | — | −0.010 | 46.5% | 0.727 | — |
| Reversal (fee) | 197 | −$21,972.75 | +$3,513.34 | $25,310.57 | 0.030 | 16.8% | 0.207 | 22.0% |
| Reversal (nofee) | 197 | +$3,465.37 | +$3,513.34 | — | 0.046 | 47.2% | 1.367 | — |
| Buy & Hold BTC | — | −$281.50 | — | — | −0.024 | — | — | 1.25% |

> Note: `exp=48.5%`, `avg_win=$117.54`, `avg_loss=−$192.97`, `payoff=0.609` for momentum; `num_long=198` (100% of trades).

---

## Interpretation

- **Reversal has positive gross edge**: +$3,513 without fees, PF=1.367. Mean-reversion in the BTC/ETH ratio works — when BTC weakens vs ETH, it tends to revert.
- **Momentum has no edge**: gross is negative (−$3,766), PF=0.727. BTC outperforming ETH does *not* predict continuation — a rising ratio is not a momentum signal.
- **The problem is turnover**: 197–198 trades across ~1724 15m candles = a trade every ~8.7 candles (~2h). Each trade costs ~$130 in fees → $25k total fees.
- **Reversal with fees is negative**: $25k in fees wipes out the $3.5k edge. To be profitable, turnover must be reduced drastically.
- **Comparison with B&H**: Buy & Hold BTC lost only $281 (−0.28%) with 1.25% DD. The strategies churn far more and lose far more.
- **Conclusion**: the cross-asset signal has predictive value (reversal), but the 15m implementation with a short lookback generates unsustainable churn. Needs a volatility filter or larger threshold.

---

## Variations

- **Larger lookback**: replace `ind["btc"][5]` with `ind["btc"][20]` or `ind["btc"][50]` — fewer trades, lower fees, potentially better signal-to-noise ratio.
- **ATR filter**: only enter when `atr[0] > avg(atr, 20)` — avoids trading in low-volatility regimes where the spread means less.
- **Long/short both legs**: when ratio rises → long ETH/short BTC; when it falls → long BTC/short ETH. Captures the bilateral spread, but requires margin.
- **Swap the asset**: use `SOLUSDT` instead of ETH. SOL has lower correlation with BTC, potentially more relative edge (but also more idiosyncratic risk).
