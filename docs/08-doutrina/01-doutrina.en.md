# The CT Doctrine — Suggested Method (Seed v3)

> **Not a verdict; it's the CT method — suggested and forkable.** Nothing here is immutable. This is a starting point, served at `ct://doutrina`.

## Axioms

1. **Epistemological (the root).** The future is **unknowable**. An indicator is just a feature of the past — it represents what already happened, it doesn't predict.
2. **Operational (corollary).** **Survival before edge.** Since prediction is impossible and error is inevitable, **risk is the foundation**; the indicator never guarantees — it only shifts odds above baseline.
3. **Mathematical (the limit).** On a path without structure (martingale), **E[P&L] = 0 for any policy**. Every method must declare which structure it harvests and measure it before building (`ct_medir_estrutura`).

## Premises

- **Trades will go wrong.** Accepting this is a premise, not a failure to eliminate.
- **The foundation survives arbitrary side.** N moments, each firing buy AND sell with the same manager — the directional term cancels; what's left is execution.
- **Loss is realized, never hidden.** Every position carries an explicit exit.
- **The stop buys the asymmetry.** It's what makes it possible to always lose little and sometimes gain much.
- **The setup exceeds the foundation.** `Net(setup)` > `Net(floor)` in the same test — otherwise the reading didn't pay.

## Beliefs (the way of thinking)

- **Imbalance is the primitive.** Force = `(a − b) / (|a| + |b|)`, signed in `[-1,+1]`.
- **Activity is the clock.** Measure force where the market transacts (VWMA by volume), not in uniform time.
- **Measure the fact, not the mean.** Primary feature is a sliding window fact — extremes, points and order.
- **Force is regime.** The same event changes meaning by level (noise/pullback/reversal).
- **Position is not outcome.** State is probabilistic context, not destiny — traps are real.
- **The ruler comes from the phenomenon.** Anchors and scale come from the window itself — the grid reacts to volatility.
- **Manage, don't predict.** The adaptive layer doesn't create EV — it improves distribution shape.
- **Predict regime, not price** — and that's SETUP, not foundation.
- **A thesis only matters if it survives the data.** Robustness before declaring.

## Method (the phases)

1. **Foundation** — prove the floor: the manager over the group survives arbitrary side (`ct_testar_sobrevivencia`).
2. **Setup** — the reading shifts direction above chance, on top of the floor.
3. **Optimization** — tune the whole system together (always with holdout and robustness).

## Validation (the cycle)

- **Unit:** the pair test — N moments, each firing buy AND sell.
- **Verdict:** Σ net (no fees) ≥ 0 in aggregate.
- **Diagnosis on failure:**
  - Floor doesn't pass → execution/management or asset without structure.
  - Reading doesn't beat baseline → keep the baseline.
  - Floor OK but setup doesn't beat → adjust usage or change the reading.
- **Triggers:** any change re-triggers the test.

---

> Next: [The survival test](./02-teste-sobrevivencia.en.md)
