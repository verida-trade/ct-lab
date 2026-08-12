# Recipe 08 — Microstructure: TFI + Backtest

> **Level:** Advanced · **Premium:** No · **Prerequisites:** Recipes 01-03, Binance account connected

## What is TFI?

- **Taker Flow Imbalance** measures the imbalance between taker buy and taker sell in the order book.
- Values **> 0** indicate buying pressure (taker buy dominates); **< 0** selling pressure.
- Unlike OHLCV, it captures real *order flow* — who is aggressively hitting the book.
- Combined with OBI and BFI, it forms the CT Lab microstructure triad.

---

## Step 1 — Collect trades

Collect real trades from Binance (3-day backfill):

```json
{ "name": "coletar_trades", "arguments": { "provider": "binance", "symbol": "BTCUSDT", "backfill_dias": 3 } }
```

Check collection status:

```json
{ "name": "coletas_ativas", "arguments": {} }
```

Wait until the collection completes. Results are saved to `ct://series/binance/BTCUSDT/trades_1s`.

---

## Step 2 — Compute ct_tfi

Compute TFI on the 1-second trades series, with a 60-second period:

```json
{ "name": "ct_tfi", "arguments": { "uri": "ct://series/binance/BTCUSDT/trades_1s", "period": 60 } }
```

**Real result (prior session):**

| Bucket (1m) | n_trades | qty | qty_delta | TFI | price |
|---|---|---|---|---|---|
| 1 | 142 | 3.8 | +1.2 | +0.32 | 118,420 |
| 2 | 98 | 2.1 | -0.8 | -0.19 | 118,398 |
| ... | ... | ... | ... | ... | ... |
| 11 | 156 | 4.2 | +0.5 | +0.14 | 118,435 |

**Divergence detected** at `1786505315`: negative TFI + positive OBI → absorption (sellers hitting but price doesn't drop).

---

## Step 3 — Aggregate TFI to 15m

TFI was computed on `trades_1s`; the backtest runs on 15m. Use `buscar_binance` + `compor_serie` to align:

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{ "name": "compor_serie", "arguments": {
    "base": "ct://series/binance/BTCUSDT/15m",
    "anexa": "ct://derived/btc_15m_tfi",
    "nome": "btc_15m_tfi_composed"
}}
```

> `compor_serie` aligns by timestamp. If timeframes are not directly compatible, it aggregates automatically.

---

## Step 4 — Backtest with TFI

Use TFI as a direction indicator in the backtest:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"tfi\"][0] > 0.3 { comprado(1.0) } else if ind[\"tfi\"][0] < -0.3 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_15m_tfi",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "tfi_strategy"
  }
}
```

---

## Real result (BOP as proxy)

BOP backtest (same microstructure family) on 15m:

| Config | Result | PF | Win Rate |
|---|---|---|---|
| ct_bop(14) **no fee** | +$398 | 1.06 | 35.7% |
| ct_bop(14) **with fee** 0.1% | **-$16,178** | — | — |

> Fees ($16,576) completely destroy the edge — the signal exists, but doesn't survive transaction costs on 15m.

---

## Interpretation

- **Microstructure captures what OHLCV can't see**: order flow, aggression, absorption.
- Indicators like TFI, OBI, and BFI reveal *who* is buying/selling, not just *the price*.
- The edge exists in the right direction — the 1.06 PF without fees proves the signal has value.
- **But on 15m with standard fees, transaction costs devour the profit** — the strategy trades too frequently.
- TFI is **most useful for real-time execution** (entry/exit timing) rather than as a backtested signal on a high timeframe.
- On smaller timeframes (1s, 1m) or with institutional fees, the edge may survive.

---

## Variations

- **Threshold 0.2 vs 0.3**: lower values increase frequency (more trades, more fees); 0.3 is already aggressive.
- **OBI as additional filter**: only trade when TFI and OBI agree (confirm pressure with book depth).
- **TFI for sizing, not direction**: use price direction + TFI to define position size (larger when TFI confirms).
- **Test 1m timeframe**: dropping to 1m can reduce fee impact per trade, increasing the edge's survival rate.
