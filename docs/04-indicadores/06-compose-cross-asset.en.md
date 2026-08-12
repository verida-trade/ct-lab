# Cross-Asset Compose

> Join series from different assets by timestamp. Essential for cross-asset analysis, spreads, and correlations.

`compor_serie` does an **inner-join** of N raw series by timestamp. The result is a **synthetic** series with columns from multiple assets aligned.

---

## When to use

| Situation | Example |
|---|---|
| Spread between assets | BTC close − ETH close |
| Ratio between assets | BTC / ETH |
| Rolling correlation | 30-bar correlation between BTC and gold |
| Pair analysis | EUR/USD vs GBP/USD |
| Multi-asset ML features | BTC + ETH + SOL features in the same dataset |

---

## Basic call

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "btc_eth_15m",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc_close" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth_close" }
    ]
  }
}
```

**Return:**
```json
{
  "uri": "ct://derived/btc_eth_15m",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

---

## The `anchor` parameter

The anchor defines the **timeline** of the output:

- **1 raw among sources:** anchor is auto-inferred
- **>1 raw among sources:** anchor is mandatory

Whenever you compose series from different assets, there will be >1 raw — anchor is mandatory. Choose the series with more data (or the most relevant one) as anchor.

---

## Example: BTC-ETH Spread

### Step 1 — Compose

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "spread_btc_eth",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth" }
    ]
  }
}
```

### Step 2 — Calculate spread via Rhai

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://derived/spread_btc_eth",
    "name": "spread_value",
    "receita": "btc - eth"
  }
}
```

### Step 3 — Calculate ratio

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://derived/spread_btc_eth",
    "name": "ratio_btc_eth",
    "receita": "btc / eth"
  }
}
```

---

## Example: 3 assets aligned

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "tri_crypto",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth" },
      { "source_uri": "ct://series/binance/SOLUSDT/15m", "source_column": "close", "as_column": "sol" }
    ]
  }
}
```

Then compute features on the synthetic series:

```rhai
// BTC-ETH correlation (approximate)
let btc_r = (btc - avg(btc));
let eth_r = (eth - avg(eth));
btc_r * eth_r
```

---

## Compose inside pipeline

You can also do compose inside the pipeline using the `compose` op:

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "multi_asset_signal",
  "output": "$signal",
  "steps": [
    { "id": "btc", "op": "sma", "source": "$anchor", "period": 20 },
    { "id": "eth", "op": "sma", "source": "ct://series/binance/ETHUSDT/15m", "period": 20 },
    {
      "id": "merge",
      "op": "compose",
      "columns": [
        { "source": "$btc", "source_column": "sma", "as_column": "btc_sma" },
        { "source": "$eth", "source_column": "sma", "as_column": "eth_sma" }
      ]
    },
    { "id": "signal", "op": "custom", "script": "if ent[\"btc_sma\"] > ent[\"eth_sma\"] { 1.0 } else { -1.0 }", "entradas": [{"alias":"btc_sma","fonte":"$merge","coluna":"btc_sma"},{"alias":"eth_sma","fonte":"$merge","coluna":"eth_sma"}], "coluna_saida": "signal" }
  ]
}
```

> **Note:** to reference an external URI in a step's `source`, use the full URI instead of `$anchor`.

---

> Next: [Custom indicators](./09-indicadores-custom.en.md) · [Declarative pipeline](./03-pipeline-declarativo.en.md)
