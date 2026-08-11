# Recipe 12 — Cross-Asset Spread (BTC vs ETH)

> **Level:** Intermediate

Align BTC and ETH, compute spread and use in backtest.

## Step 1 — Fetch both series

```json
[
  { "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } },
  { "name": "buscar_binance", "arguments": { "symbol": "ETHUSDT", "interval": "15m" } }
]
```

## Step 2 — Compose

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "btc_eth_spread",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth" }
    ]
  }
}
```

## Step 3 — Compute spread and ratio

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://derived/btc_eth_spread",
    "name": "btc_eth_ratio",
    "receita": "#{\"spread\": btc - eth, \"ratio\": btc / eth}"
  }
}
```

## Step 4 — Backtest

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"ratio\"][0] > ind[\"ratio\"][5] { comprado(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_eth_ratio",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "cross_asset"
  }
}
```

> Strategy: long BTC when BTC/ETH ratio is rising (BTC gaining relative strength).
