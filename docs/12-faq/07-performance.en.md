# FAQ — Performance

## Too many series in cache

If your strategy uses many indicators, the cache (100 premium series) may fill up.

**Solution:** Use `indicadores_receitas` (inline) instead of pre-materializing — recipes are computed on-the-fly and don't occupy a cache slot.

## Slow backtest

- Reduce `num_trades` by simplifying the strategy
- Use larger timeframes (15m instead of 1m)
- Inline indicators (`indicadores_receitas`) are faster than pre-materialize + URI

## High-volume microstructure collection

100 symbols × ~6000 msg/s peak — consider reducing symbol count if CPU/IO limited.
