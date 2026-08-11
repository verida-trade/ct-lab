# 05 · Series Composition

Series composition is the feature that lets you **combine multiple OHLCV series into a single derived series**, aligning them by timestamp (inner-join). This is essential for cross-asset analysis, correlation, spread trading, and feeding ML models with multivariate features.

CT Lab provides the `compor_serie` tool for this purpose.

---

## The Problem Composition Solves

Imagine you want to analyze the relationship between BTC and ETH. You have:

- `ct://series/binance/BTCUSDT/15m` — 1000 candles
- `ct://series/binance/ETHUSDT/15m` — 1000 candles

But each series lives in isolation. To compute the spread `btc_close - eth_close` or the correlation between returns, you need **both series aligned by timestamp** in a single structure.

Composition does exactly this: an **inner-join** by timestamp that produces a derived series with columns from multiple sources.

---

## `compor_serie` — Inner-Join by Timestamp

### The *Anchor* Concept

The derived series needs a **temporal reference** — the *anchor*. The anchor defines which timestamps the derived series will contain.

| Scenario | Anchor Rule |
|----------|-------------|
| 1 raw series as source | Auto-inferred: the anchor is the raw series itself |
| >1 raw series as sources | **Explicit** anchor required: pick one raw series as anchor |

The anchor determines the timestamp axis. All other series are aligned (inner-joined) against the anchor. Only timestamps present in **all** sources are kept.

---

### AI Chat Prompt

> "Create a series combining BTC's close and ETH's close, aligned by timestamp, with BTC as the anchor."

### Tool Call

```json
{
  "tool": "compor_serie",
  "arguments": {
    "name": "btc_eth",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      {
        "source_uri": "ct://series/binance/BTCUSDT/15m",
        "source_column": "close",
        "as_column": "btc_close"
      },
      {
        "source_uri": "ct://series/binance/ETHUSDT/15m",
        "source_column": "close",
        "as_column": "eth_close"
      }
    ]
  }
}
```

### Expected Return

```json
{
  "uri": "ct://derived/btc_eth",
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|:--------:|-------------|
| `name` | `string` | ✅ | Name of the derived series (becomes part of the URI) |
| `anchor` | `string` | ✅ | URI of the raw series serving as temporal axis |
| `columns` | `array` | ✅ | List of column mappings: source → destination |
| `columns[].source_uri` | `string` | ✅ | URI of the source series |
| `columns[].source_column` | `string` | ✅ | Column in the source series (`open`, `high`, `low`, `close`, `volume`) |
| `columns[].as_column` | `string` | ✅ | Column name in the derived series |

### Return

| Field | Description |
|-------|-------------|
| `uri` | URI of the derived series: `ct://derived/<name>` |
| `row_count` | Number of rows after the inner-join |
| `first_ts` / `last_ts` | Temporal bounds |
| `anchor_uri` | URI of the anchor series |

> **Note:** `row_count` may be **smaller** than the candle count of any individual series, since the inner-join keeps only timestamps present in **all** sources.

---

## Complete Example: BTC/ETH Spread

Let's build a complete end-to-end example: ingest BTC and ETH, compose them, and verify the result.

### Step 1: Ingest BTCUSDT

```json
{
  "tool": "buscar_binance",
  "arguments": { "symbol": "BTCUSDT", "interval": "15m" }
}
```

Return:
```json
{ "uri": "ct://series/binance/BTCUSDT/15m", "row_count": 1000 }
```

### Step 2: Ingest ETHUSDT

```json
{
  "tool": "buscar_binance",
  "arguments": { "symbol": "ETHUSDT", "interval": "15m" }
}
```

Return:
```json
{ "uri": "ct://series/binance/ETHUSDT/15m", "row_count": 1000 }
}
```

### Step 3: Compose the Series

```json
{
  "tool": "compor_serie",
  "arguments": {
    "name": "btc_eth_spread",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      {
        "source_uri": "ct://series/binance/BTCUSDT/15m",
        "source_column": "close",
        "as_column": "btc_close"
      },
      {
        "source_uri": "ct://series/binance/ETHUSDT/15m",
        "source_column": "close",
        "as_column": "eth_close"
      }
    ]
  }
}
```

Return:
```json
{
  "uri": "ct://derived/btc_eth_spread",
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

> 950 rows — the 50 difference is timestamps where ETH had no matching candle.

### Step 4: Read the Data via Resource

The derived series can be read like any other series:

```
ct://derived/btc_eth_spread/tail/5
```

Response (resource):
```json
{
  "uri": "ct://derived/btc_eth_spread/tail/5",
  "row_count": 5,
  "columns": ["btc_close", "eth_close"],
  "timestamps": [
    "2026-08-11T14:00:00Z",
    "2026-08-11T14:15:00Z",
    "2026-08-11T14:30:00Z",
    "2026-08-11T14:45:00Z",
    "2026-08-11T15:00:00Z"
  ],
  "rows": [
    { "btc_close": 64000.0, "eth_close": 3200.0 },
    { "btc_close": 64100.0, "eth_close": 3210.0 },
    { "btc_close": 63950.0, "eth_close": 3185.0 },
    { "btc_close": 64050.0, "eth_close": 3195.0 },
    { "btc_close": 64080.0, "eth_close": 3205.0 }
  ]
}
```

### Step 5: Verify with `info_serie`

```json
{
  "tool": "info_serie",
  "arguments": { "uri": "ct://derived/btc_eth_spread" }
}
```

Return:
```json
{
  "uri": "ct://derived/btc_eth_spread",
  "kind": "derived",
  "columns": ["btc_close", "eth_close"],
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "last_accessed_at": "2026-08-11T15:12:00Z",
  "source_uris": [
    "ct://series/binance/BTCUSDT/15m",
    "ct://series/binance/ETHUSDT/15m"
  ]
}
```

> Note that `kind` is `"derived"` and `source_uris` lists the original series.

---

## Adding More Columns

You can pull multiple columns from the same series:

```json
{
  "tool": "compor_serie",
  "arguments": {
    "name": "btc_eth_full",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close",  "as_column": "btc_close" },
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "volume", "as_column": "btc_volume" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close",  "as_column": "eth_close" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "volume", "as_column": "eth_volume" }
    ]
  }
}
```

Return:
```json
{
  "uri": "ct://derived/btc_eth_full",
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

---

## TypeScript Example

```typescript
// Ingest both series
await Ct.buscarBinance({ symbol: "BTCUSDT", interval: "15m" });
await Ct.buscarBinance({ symbol: "ETHUSDT", interval: "15m" });

// Compose
const composed = await Ct.comporSerie({
  name: "btc_eth_spread",
  anchor: "ct://series/binance/BTCUSDT/15m",
  columns: [
    {
      sourceUri: "ct://series/binance/BTCUSDT/15m",
      sourceColumn: "close",
      asColumn: "btc_close",
    },
    {
      sourceUri: "ct://series/binance/ETHUSDT/15m",
      sourceColumn: "close",
      asColumn: "eth_close",
    },
  ],
});
console.log(`Derived series: ${composed.uri}`);
console.log(`${composed.row_count} aligned rows`);
```

---

## Use Cases

| Use Case | How Composition Helps |
|----------|----------------------|
| Spread trading (BTC-ETH) | Aligns closes of two assets in one series |
| Multi-asset correlation | Combines N closes for correlation matrix |
| ML features | Joins features from multiple series into a derived one |
| Pair trading | Aligns prices to compute ratio and deviation |

---

[← Back to category 03](README.en.md)
