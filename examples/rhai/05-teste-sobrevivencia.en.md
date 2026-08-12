# Recipe 05 — Survival Test (Grid)

> **Level:** Intermediate · **Plan:** Premium
> **Prerequisite:** Read [`docs/08-doutrina/`](../../docs/08-doutrina/) — understand the concept of "random side" and why a profitable system must survive the floor of chance.
> **Series source:** `ct://series/binance/BTCUSDT/15m` (1724 candles)

---

## What is it?

- The **Survival Test** fires a buy **and** a sell at the **same** moment, with the **same** risk manager. It measures how negative the floor of arbitrary side choice is.
- If the system has no entry _edge_, the average pair result (buy + sell) should tend to zero **before** costs. Any negative value is pure execution "bleed."
- The key metric is **EV_par_reguas**: the expected value of each pair **normalized by the ruler** (stop→activation distance). `EV ≈ 0` means a neutral floor; `EV ≪ 0` means bleeding.
- It is the cruelest test there is: it doesn't ask "did you get the side right?" — it asks "how much do you lose just by _entering_?"

---

## Step 1 — Fetch series

```json
{ "name": "buscar_binance", "arguments": { "par": "BTCUSDT", "tempo": "15m" } }
```

The series becomes available at `ct://series/binance/BTCUSDT/15m`.

---

## Step 2 — (Optional) Measure structure

```json
{ "name": "ct_medir_estrutura", "arguments": { "serie": "ct://series/binance/BTCUSDT/15m" } }
```

Returns the typical volatility ruler of the asset, useful for calibrating `stop_r` and `dist_r`.

---

## Step 3 — Run the test

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

| Parameter      | Value | Meaning                                           |
|----------------|-------|---------------------------------------------------|
| `momentos`     | 20    | How many time points fire the buy+sell pair        |
| `fee_pct`      | 0     | fee per trade as percent (0 = no cost)            |
| `stop_r`       | 0.5   | stop in multiples of the ruler                    |
| `ativacao_r`   | 1.0   | activation (target) in multiples of the ruler     |
| `dist_r`       | 0.3   | entry distance in multiples of the ruler          |
| `prazo`        | 128   | max candles per trade                              |
| `breakeven`    | true  | move stop to breakeven after activation           |
| `reescala_vol` | true  | normalize stop/target by local volatility         |
| `piramide`     | false | no added lots                                     |

The response contains `n_momentos` entries, each with: `ts`, `regua`, `pnl_compra`, `pnl_venda`, `par_reguas`.

---

## Real result — 3 variants

We ran three variants on the same series (BTCUSDT 15m, 1724 candles, 20 moments):

| Variant              | fee_pct | piramide | soma_pnl      | soma_par_reguas | EV_par_reguas | pares_positivos |
|----------------------|---------|----------|---------------|-----------------|---------------|-----------------|
| ① No fee, no pir.    | 0       | false    | **−$1,004.03**| −0.777          | **−0.0388**   | **12 / 20**      |
| ② With fee 0.1 %     | 0.1     | false    | −$6,132.61    | −8.13           | −0.407        | 4 / 20           |
| ③ No fee, with pir.  | 0       | true     | −$2,311.48    | −3.43           | −0.171        | 11 / 20          |

**Example moments (variant ①):**

| ts          | ruler   | pnl_buy   | pnl_sell  | par_reguas |
|-------------|---------|-----------|-----------|------------|
| 1785447900  | 1233.31 | 0.00      | −616.65   | −0.500     |
| 1785491100  | 1771.38 | −899.79   | 1007.85   | +0.061     |
| 1785753900  | 1496.33 | +997.38   | −748.17   | +0.167     |

---

## Interpretation

- **EV_par_reguas = −0.0388 (no fee)** → each pair loses ~3.9 % of the ruler. It is a small but **negative** bias — execution bleeds, even if only slightly, from arbitrary side choice.
- **12 / 20 positive pairs (no fee)** → the floor is _marginal_. Execution doesn't bleed much: in 60 % of moments, a buy+sell pair closes green. This means the _edge_ must come from the **entry**, not the execution.
- **With fee 0.1 %: only 4 / 20 positive pairs, EV = −0.407** → catastrophic. The fee turns 12/20 into 4/20. Transaction cost destroys the neutral floor. Conclusion: the _setup_ must produce _edge_ via entry, not via execution.
- **Pyramid makes it worse: −$2,311 vs −$1,004** → adding size does not improve the _edge-to-cost_ ratio. Pyramid is leverage on a floor that is already marginal.
- **PnL scale:** the ruler ranges from ~1,233 to ~1,771 (locally rescaled volatility). The $1,004 loss over 20 pairs is ~$50/pair — a small but consistent drag.
- **Bottom line:** the Survival Test doesn't tell you _whether_ you make money; it tells you _how much_ the mere act of entering costs. If the floor is negative, any strategy needs an entry _edge_ larger than the bleed measured here.

---

## Variations

- 🔧 **Change `stop_r` / `ativacao_r` / `dist_r`** → A wider ruler (`stop_r = 1.0`) reduces the relative fee cost; a shorter activation (`ativacao_r = 0.5`) brings the exit closer and reduces floor exposure.
- 📈 **`piramide = true`** → Reconsider only if your _setup_ has a proven positive _edge_; pyramid only makes sense when the floor is already positive.
- ⏱️ **Different _timeframe_** (`5m`, `1h`) → Changes the typical ruler and the number of usable moments. Longer series tend to reduce floor noise.
- 🔢 **More moments (`momentos = 50`)** → Reduces the variance of the EV estimate; recommended before declaring a floor "real."
