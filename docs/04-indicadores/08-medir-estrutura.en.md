# Measuring Structure — `ct_medir_estrutura`

> **Premium.** Measures variance ratio and kurtosis by scale and volatility regime **before** trading. Answers: "is there structure to harvest in this asset?"

Before building any strategy, it's fundamental to know if the asset has **structure** — that is, if the price path has exploitable properties (mean-reversion, momentum, etc.). `ct_medir_estrutura` answers this with rigorous metrics.

---

## What it measures

### Variance Ratio (VR)

`VR(τ) = Var(r_τ) / (τ · Var(r_1))`

| VR | Interpretation |
|---|---|
| VR ≈ 1 | Random walk (martingale) — **no structure** |
| VR < 1 | Anti-persistence (mean-reversion) — oscillates, reverts |
| VR > 1 | Persistence (momentum) — trends, walks |

### Kurtosis

Measures the "tail" of return distribution. High kurtosis = rare but extreme events (crashes, rebounds).

---

## Call

```json
{
  "name": "ct_medir_estrutura",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m"
  }
}
```

**Return (summary format):**
```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "blocos": 8,
  "escalas": [1, 5, 15, 60, 240],
  "vr_por_escala": { ... },
  "curtose_por_escala": { "1": 72.3, "5": 45.1, ... },
  "vr_por_regime": {
    "baixa": { "vr_1": 1.5, "vr_16": 1.8, "vr_64": 2.0 },
    "alta": { "vr_1": 0.71, "vr_16": 0.59, "vr_64": 0.49 }
  },
  "veredito": {
    "vol_alta": "anti-persistente_estavel (VR<1 in 8/8 blocks)",
    "vol_baixa": "persistente (VR 1.2-2.0)"
  }
}
```

---

## How to interpret

### Example: BTCUSDT 15m

| Vol regime | VR(τ=1) | VR(τ=16) | VR(τ=64) | Interpretation |
|---|---|---|---|---|
| **High vol** | 0.71 | 0.59 | 0.49 | Anti-persistent stable — reverts. 8/8 blocks VR<1 |
| **Low vol** | 1.5 | 1.8 | 2.0 | Persistent — walks. Mean-reversion fails |

**Conclusion:** In high vol, BTCUSDT 15m oscillates and reverts (VR<1). In low vol, it walks (VR>1). Structure exists, but is **regime-dependent**.

### Rules of thumb

| Situation | VR | Action |
|---|---|---|
| VR ≈ 1 in all regimes | No structure | Nothing to harvest — find another asset |
| VR < 1 (anti-persistence) | Mean-reversion | Reversal strategies may work |
| VR > 1 (persistence) | Momentum | Trend strategies may work |
| Very high kurtosis | Fat tails | Stops are essential — extreme events happen |

---

## Why this matters

CT doctrine states: **on a path without structure (martingale), E[P&L] = 0 for any policy**. Before optimizing any strategy, measure structure. If there's no structure, no indicator will generate edge — it's self-deception.

> See also: [CT Doctrine — Mathematical Axiom](../08-doutrina/)

---

> Next: [Custom indicators](./09-indicadores-custom.en.md) · [Cookbook](./07-cookbook.en.md)
