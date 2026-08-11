# Receita 12 — Cross-Asset Spread (BTC vs ETH)

> **Nível:** Intermediário

Alinhar BTC e ETH, calcular spread e usar em backtest.

## Passo 1 — Buscar ambas as séries

```json
[
  { "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } },
  { "name": "buscar_binance", "arguments": { "symbol": "ETHUSDT", "interval": "15m" } }
]
```

## Passo 2 — Compor

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

## Passo 3 — Calcular spread e razão

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://derived/btc_eth_spread",
    "name": "btc_eth_ratio",
    "receita": "#{\"spread\": btc - eth, \"ratio\": btc / eth, \"log_ratio\": log(btc / eth)}"
  }
}
```

## Passo 4 — Backtest

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

> Estratégia: long BTC quando a razão BTC/ETH está subindo (BTC ganhando força relativa).
