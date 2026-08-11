# Guardrails — Anti Self-Deception

> Lessons paid with error in the grid investigation. Robustness checklist before declaring any result.

## Robustness checklist

Before declaring a strategy "works", verify:

- [ ] **Untouched holdout:** train and validation separated temporally, no leakage
- [ ] **Walk-forward:** multiple validation windows, not just one
- [ ] **Parameter surface:** result isn't an isolated peak — there's a contiguous, monotonic region
- [ ] **Cross-TF:** result generalizes to another timeframe
- [ ] **Realistic cost:** fee ≠ 0 (0.04% typical); turnover often kills
- [ ] **Out-of-sample:** regime reading used as setup was tested outside the training sample

## Known traps

| Trap | How it appears | How to avoid |
|---|---|---|
| **Lookahead** | Viterbi/predict_proba use the entire sequence | Use causal version (predict last) |
| **Timeout drift** | "Close at market at test timeout" credits drift | Close at the real bar price |
| **Biased sample** | The "24 largest episodes" tell a different story | Test on aggregate, not selected |
| **Composite metric hides** | ct_regime lag 0 was a spike | Look at individual metrics |
| **Strategy carries state** | `EstrategiaRhai` carries state between backtests | Recompile per backtest |
| **Fee=0 deceives** | Pure directional: 9.3× gross → 0.94× at 0.04% | Always test with realistic fee |

---

> Back to: [README](./README.en.md)
