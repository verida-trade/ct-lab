# The Simulator & Accounting

> How the backtest engine simulates execution, OCO, fees and turnover.

## Execution model

At each bar, the engine:

1. Receives the strategy's decision (Alvo: long/short/flat)
2. If position changed, executes the trade at the bar's price
3. Updates the equity curve

### Execution price

- **Entry:** close of the bar where Alvo changed
- **Stop/Limit (via grupo):** if the bar's high/low touches the level, it executes
- **Pessimistic OCO:** if multiple exits are touched in one candle, only the pessimistic one executes (conservative modeling under OHLC ambiguity)

### Fee

`fee_pct` = fraction of notional per trade. E.g.: 0.001 = 0.1%.

```
fee = |qty × price| × fee_pct
```

Fee is charged on **every trade** (entry and exit).

---

## Turnover

Turnover = sum of notional across all trades. If the strategy trades a lot (high turnover), fees eat the edge:

- Pure directional strategy: 9.3× gross → 0.94× at 0.04% fee
- Always compare `pnl_bruto` vs `pnl_total` in the result

---

> Next: [Metrics](./12-metricas.en.md)
