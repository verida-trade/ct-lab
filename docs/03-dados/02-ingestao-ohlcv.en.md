# 02 · OHLCV Ingestion

Ingestion is the act of fetching candlestick data (Open-High-Low-Close-Volume) from an external provider and storing it in CT Lab's local cache. Every ingested series receives a **canonical URI** that can be referenced by indicators, pipelines, and ML models.

CT Lab offers four ingestion tools:

| Tool | Description |
|------|-------------|
| `buscar_serie` | Generic — dispatches to the correct provider based on the `provider` parameter. |
| `buscar_binance` | Shortcut for Binance (equivalent to `buscar_serie` with `provider: "binance"`). |
| `buscar_yahoo` | Shortcut for Yahoo Finance. |
| `importar_csv` | Imports data from a custom CSV file. |

---

## The `IngestResult` Shape

All ingestion tools return an object in the same format — called `IngestResult`:

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-11T10:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": []
}
```

| Field | Description |
|-------|-------------|
| `uri` | Canonical URI of the series in the local repository |
| `row_count` | Number of candles ingested |
| `first_ts` | Timestamp of the first candle |
| `last_ts` | Timestamp of the last candle |
| `evicted_series` | List of URIs removed by LRU eviction (cache full) |

> **Note:** The return contains metadata only. Point-by-point data is read via **resources** using URIs like `ct://series/binance/BTCUSDT/15m/tail/10`. See [06-data-uris](06-uris-dados.en.md) for details.

---

## `buscar_serie` — Generic Ingestion

The generic tool dispatches to the correct provider based on `provider`.

### AI Chat Prompt

> "Fetch 1000 candles of BTCUSDT at 15-minute intervals from Binance."

### Tool Call

```json
{
  "tool": "buscar_serie",
  "arguments": {
    "provider": "binance",
    "symbol": "BTCUSDT",
    "interval": "15m"
  }
}
```

### Expected Return

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": []
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|:--------:|-------------|
| `provider` | `string` | ✅ | `"binance"`, `"yahoo"`, etc. |
| `symbol` | `string` | ✅ | Trading pair (e.g., `"BTCUSDT"`) or ticker (e.g., `"AAPL"`) |
| `interval` | `string` | ✅ | Candle interval: `"1m"`, `"5m"`, `"15m"`, `"1h"`, `"4h"`, `"1d"`, etc. |
| `limit` | `number` | ❌ | Maximum number of candles (default varies by provider) |
| `until` | `string` | ❌ | ISO 8601 timestamp — fetch candles **before** this date |

---

## `buscar_binance` — Binance Shortcut

Equivalent to `buscar_serie` with `provider` fixed to `"binance"`. One fewer parameter to type.

### AI Chat Prompt

> "Download BTCUSDT 15m candles from Binance."

### Tool Call

```json
{
  "tool": "buscar_binance",
  "arguments": {
    "symbol": "BTCUSDT",
    "interval": "15m"
  }
}
```

### Expected Return

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": []
}
```

---

## `buscar_yahoo` — Yahoo Finance Shortcut

For stocks, ETFs, and other traditional instruments.

### AI Chat Prompt

> "Fetch daily candles for AAPL from Yahoo Finance."

### Tool Call

```json
{
  "tool": "buscar_yahoo",
  "arguments": {
    "symbol": "AAPL",
    "interval": "1d"
  }
}
```

### Expected Return

```json
{
  "uri": "ct://series/yahoo/AAPL/1d",
  "row_count": 1000,
  "first_ts": "2022-09-01T00:00:00Z",
  "last_ts": "2026-08-11T00:00:00Z",
  "evicted_series": []
}
```

---

## The `until` Parameter — Scroll-Back Pagination

The `until` parameter lets you fetch candles **before a specific date**. This is essential for paginating large volumes of historical data without repeating data.

### How It Works

1. First call: no `until` → returns the most recent candles.
2. Use `first_ts` from the result as `until` in the next call.
3. Repeat until you reach the desired depth.

### AI Chat Prompt

> "Fetch BTCUSDT 15m candles up to August 1st, 2026."

### Tool Call

```json
{
  "tool": "buscar_binance",
  "arguments": {
    "symbol": "BTCUSDT",
    "interval": "15m",
    "until": "2026-08-01T00:00:00Z"
  }
}
```

### Expected Return

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-07-31T10:00:00Z",
  "last_ts": "2026-08-01T00:00:00Z",
  "evicted_series": []
}
```

### Full Pagination Pattern

```typescript
// Scroll-back pagination: fetch 5000 candles in batches of 1000
let allFirstTs: string | undefined;
const batches = [];

for (let i = 0; i < 5; i++) {
  const result = await Ct.buscarBinance({
    symbol: "BTCUSDT",
    interval: "15m",
    until: allFirstTs,
  });
  batches.push(result);
  // Use first_ts as cursor for the next batch
  // (In practice, read first_ts from the stored series)
  if (result.row_count < 1000) break; // no more data
}
```

---

## `importar_csv` — Custom CSV Import

Bring your own data. Useful for exotic assets, synthetic data, or custom spreadsheets.

### AI Chat Prompt

> "Import the file ~/data/my_custom_data.csv as a series with symbol CUSTOM1 at 1h interval."

### Tool Call

```json
{
  "tool": "importar_csv",
  "arguments": {
    "path": "~/data/my_custom_data.csv",
    "symbol": "CUSTOM1",
    "interval": "1h",
    "provider": "custom"
  }
}
```

### Expected CSV Format

The file should contain timestamp and OHLCV columns:

```csv
timestamp,open,high,low,close,volume
2026-08-11T10:00:00Z,64000.0,64100.0,63950.0,64080.0,125.5
2026-08-11T11:00:00Z,64080.0,64200.0,64000.0,64150.0,98.3
2026-08-11T12:00:00Z,64150.0,64300.0,64100.0,64250.0,152.1
```

### Expected Return

```json
{
  "uri": "ct://series/custom/CUSTOM1/1h",
  "row_count": 3,
  "first_ts": "2026-08-11T10:00:00Z",
  "last_ts": "2026-08-11T12:00:00Z",
  "evicted_series": []
}
```

---

## Python Example with `uv`

```python
# Install dependencies
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # BTCUSDT 15m from Binance
    btc = await client.buscar_binance(
        symbol="BTCUSDT",
        interval="15m",
    )
    print(f"BTC: {btc['row_count']} candles"
          f" ({btc['first_ts']} → {btc['last_ts']})")
    print(f"URI: {btc['uri']}")

    # AAPL 1d from Yahoo
    aapl = await client.buscar_yahoo(
        symbol="AAPL",
        interval="1d",
    )
    print(f"AAPL: {aapl['row_count']} candles")
    print(f"URI: {aapl['uri']}")

    # Scroll-back pagination
    older = await client.buscar_binance(
        symbol="BTCUSDT",
        interval="15m",
        until=btc["first_ts"],
    )
    print(f"Previous batch: {older['row_count']} candles")

asyncio.run(main())
```

```bash
# Run
uv run main.py
```

---

## URI Patterns by Provider

| Provider | URI Pattern |
|----------|-------------|
| Binance | `ct://series/binance/<symbol>/<interval>` |
| Yahoo | `ct://series/yahoo/<symbol>/<interval>` |
| Custom (CSV) | `ct://series/custom/<symbol>/<interval>` |

> The URI is the series' identity in the repository. All subsequent operations (indicators, composition, ML) reference the series by this URI.

---

[← Back to category 03](README.en.md)
