# The Grid Investigation

> Record of the grid investigation as risk management foundation — the v1→v9 journey refuted and the floor that stood.

## The purpose

The grid+filler **is not a strategy**: it's the **management layer** that administers any position. The test is **arbitrary-side** — if the layer doesn't bleed even with coin flips, any setup with edge becomes profit on top of the floor.

## The v1→v9 journey — all refuted

| v | Form | Verdict | Cause |
|---|---|---|---|
| v1 | Arbitrary fixed multiples | ✗ | Period artifact |
| v2 | Excursion quantiles | ✗ | R:R≈1, no edge |
| v3 | First-passage EV | ✗ | Timeout drift, doesn't translate |
| v4 | Profile + selective gate | ✗ | 0 trades: reading never signals |
| v5 | Regime-reactive | ✗ | Instrument×signal mismatch |
| v6 | Asymmetric + trailing | ✗ | Declared "pass" early; robustness refuted |
| v7 | v6 + pyramid | ✗ | All variance from one period |
| v8 | One-shot, 3 levels | ✗ | Doesn't harvest QV |
| v9 | Continuous session | ✗ | Analytical EV ≈ −0.3r |

**Truncated mean-reversion is dead on BTC 15m** — model, measurement and engine agree.

## The floor that stood — the synthetic straddle

Buy+sell pair with **short stop + trailing with far activation** — fat tail pays for stops.

- **EV +0.03 to +0.09 régua per pair**, positive in both vol regimes and train/holdout
- Arbitrary-side floor: **EV ≥ 0 without fees — exists**. But it's thin; 0.04% fee consumes the EV
- The fat edge must come from the **setup**

## Method lessons (paid with error)

1. **Robustness before declaring** (surface + walk-forward + cross-TF + cost)
2. **Close the analytical EV with curves before running the engine**
3. **Cost is first-class** — fee=0 deceives; turnover kills
4. **Lookahead hides** — Viterbi/predict_proba use the entire sequence
5. **Biased sample deceives** — the "24 largest" tell a different story
6. **The instrument must match the signal**

---

> Next: [Forking the doctrine](./04-fork-doutrina.en.md)
