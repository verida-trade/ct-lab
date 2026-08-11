# The Survival Test

> The CT method's ruler: if execution doesn't bleed even with coin flips, any setup with edge becomes profit.

## The instrument

The "pair" = two independent executions (buy and sell) over the same N moments (20 canonical). The directional term cancels by construction — what's left measures **execution**.

## Verdict

**Σ net (no fees) ≥ 0** in aggregate = the floor exists.

With fees is the setup's real-world test.

## How to run

See [Survival Test (Grid) in the Backtest section](../05-backtest/08-teste-sobrevivencia.en.md) for the full `ct_testar_sobrevivencia` tool reference.

## Interpretation

| Result | Diagnosis |
|---|---|
| Σ ≥ 0 (no fees) | The floor exists — execution doesn't bleed |
| Σ < 0 (no fees) | Poor execution/management or asset without structure |
| Σ ≥ 0 no fees, < 0 with fees | Turnover kills — fat edge must come from setup |
| Setup doesn't beat floor | The reading brought no edge — keep the baseline |

---

> Next: [The grid investigation](./03-investigacao-grid.en.md)
