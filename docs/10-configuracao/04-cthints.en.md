# The `.cthints` File

`.cthints` is a per-project hints file that the agent reads for context:

```json
{
  "project": "my-ct-project",
  "description": "Mean-reversion strategies on BTC 15m",
  "timeframe": "15m",
  "symbols": ["BTCUSDT", "ETHUSDT"],
  "risk_tolerance": "conservative"
}
```

Place `.cthints` at your project root. The agent incorporates the context automatically.
