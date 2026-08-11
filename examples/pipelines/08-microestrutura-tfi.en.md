# Recipe 08 — Microstructure: TFI + Backtest

> **Level:** Advanced · **Premium** · **Prerequisites:** [Microstructure](../docs/07-microestrutura/)

Collect trades, compute TFI, use as signal in backtest.

## Step 1 — Collect trades

```json
{
  "name": "coletar_trades",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "backfill_dias": 3 }
}
```

Wait a few minutes to accumulate data. Check status:

```json
{ "name": "coletas_ativas", "arguments": {} }
```

## Step 2 — Compute ct_tfi

```json
{
  "name": "ct_tfi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/trades_1s", "period": 60 }
}
```

## Step 3 — Aggregate to 15m and backtest

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "btc_15m_tfi",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "close" },
      { "source_uri": "ct://derived/ct_tfi_...", "source_column": "ct_tfi", "as_column": "tfi" }
    ]
  }
}
```

## Step 4 — Backtest

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

> **Note:** TFI measures taker flow imbalance — values > 0 indicate buying pressure.
