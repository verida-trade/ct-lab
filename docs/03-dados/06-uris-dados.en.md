# 06 · Data URIs — Deep Dive

One of the most important architectural decisions in CT Lab is the **separation of metadata and data**. MCP tools (`buscar_serie`, `compor_serie`, etc.) return only **metadata** — URI, row count, temporal bounds. Point-by-point data is read via **resources** using URI templates.

This prevents bloating the AI model's context with thousands of candles. The model reads only what it needs, when it needs it.

---

## The Separation: Tools vs Resources

| Aspect | Tools (MCP) | Resources (MCP) |
|--------|-------------|-----------------|
| What they return | URI + metadata | Point-by-point data (rows) |
| When to use | Ingestion, composition, management | Reading data for analysis |
| Return size | Small (~100 bytes) | Variable (up to ~N rows) |
| Example | `buscar_serie` → `{ uri, row_count, ... }` | `ct://series/.../tail/5` → `{ rows: [...] }` |

### Why This Matters

If `buscar_serie` returned 1000 OHLCV candles in JSON, the AI model's context would be consumed quickly. Instead:
1. The tool returns only the URI and metadata (~100 bytes).
2. The model decides **how many rows** it needs and at **what position** in the dataset.
3. The model reads via resource URI (e.g., `/tail/5` to see the last 5 rows).

This is **essential for context economy** in long conversations.

---

## Resource URI Templates

### Raw Series (Ingested)

| Template | Description |
|----------|-------------|
| `ct://series/<provider>/<symbol>/<interval>/tail/<N>` | Last N rows |
| `ct://series/<provider>/<symbol>/<interval>/head/<N>` | First N rows |
| `ct://series/<provider>/<symbol>/<interval>/sample/<N>` | Random sample of N rows |

### Derived Series (Composed)

| Template | Description |
|----------|-------------|
| `ct://derived/<name>/tail/<N>` | Last N rows of derived series |
| `ct://derived/<name>/head/<N>` | First N rows |
| `ct://derived/<name>/sample/<N>` | Random sample of N rows |

### Full URI Examples

```
ct://series/binance/BTCUSDT/15m/tail/5       # Last 5 candles of BTCUSDT 15m
ct://series/binance/BTCUSDT/15m/head/10      # First 10 candles
ct://series/binance/BTCUSDT/15m/sample/20     # 20 random rows
ct://series/yahoo/AAPL/1d/tail/3              # Last 3 daily candles of AAPL
ct://derived/btc_eth_spread/tail/5            # Last 5 rows of composed series
```

---

## JSON Response from a Resource

When the AI model reads a resource URI, it receives a JSON with this shape:

### AI Chat Prompt

> "Show me the last 5 candles of BTCUSDT at 15-minute intervals."

### Resource URI Read by Model

```
ct://series/binance/BTCUSDT/15m/tail/5
```

### JSON Response

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m/tail/5",
  "row_count": 5,
  "columns": ["open", "high", "low", "close", "volume"],
  "timestamps": [
    "2026-08-11T14:00:00Z",
    "2026-08-11T14:15:00Z",
    "2026-08-11T14:30:00Z",
    "2026-08-11T14:45:00Z",
    "2026-08-11T15:00:00Z"
  ],
  "rows": [
    { "open": 63980.0, "high": 64100.0, "low": 63950.0, "close": 64000.0, "volume": 125.5 },
    { "open": 64000.0, "high": 64150.0, "low": 63980.0, "close": 64100.0, "volume":  98.3 },
    { "open": 64100.0, "high": 64200.0, "low": 63950.0, "close": 63950.0, "volume": 152.1 },
    { "open": 63950.0, "high": 64100.0, "low": 63920.0, "close": 64050.0, "volume": 110.7 },
    { "open": 64050.0, "high": 64120.0, "low": 64000.0, "close": 64080.0, "volume":  87.4 }
  ]
}
```

### Response Fields

| Field | Description |
|-------|-------------|
| `uri` | The full resource URI (including `/tail/5`) |
| `row_count` | Number of rows returned (may be < N if the series has fewer) |
| `columns` | Available column names |
| `timestamps` | Array of ISO 8601 timestamps (X axis) |
| `rows` | Array of objects with key = column name |

---

## `tail` vs `head` vs `sample`

| Operation | Position | Updates `last_accessed_at`? |
|-----------|----------|:--------------------------:|
| `tail/N` | End of series (most recent) | ✅ Yes (updates LRU) |
| `head/N` | Start of series (oldest) | ✅ Yes |
| `sample/N` | Random sample | ✅ Yes |

> All resource reads update `last_accessed_at`, affecting LRU eviction priority. Frequently-read series are protected from eviction.

---

## Derived Series — Same Pattern

A series composed via `compor_serie` produces URI `ct://derived/<name>` and follows exactly the same templates:

### Resource URI

```
ct://derived/btc_eth_spread/tail/5
```

### JSON Response

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

Note how the columns match the `as_column` defined during composition.

---

## Recommended Usage Flow

```
1. Tool: buscar_serie          → returns URI + metadata
2. Resource: ct://.../head/10  → model verifies the structure
3. Resource: ct://.../tail/10  → model sees most recent data
4. Resource: ct://.../sample/50 → model samples for statistical inspection
5. Indicator or pipeline uses the URI for computation
```

The AI model dynamically decides how many rows to read based on the task. For example, to compute a 14-period RSI, the model might read `/tail/20` (14 + margin).

---

## Python Example with `uv`

```python
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # Step 1: Ingest (tool returns metadata)
    result = await client.buscar_binance(
        symbol="BTCUSDT",
        interval="15m",
    )
    uri = result["uri"]
    print(f"Ingested: {uri} ({result['row_count']} candles)")

    # Step 2: Read last 5 rows via resource
    resource_uri = f"{uri}/tail/5"
    data = await client.read_resource(resource_uri)
    print(f"Columns: {data['columns']}")
    for i, ts in enumerate(data["timestamps"]):
        row = data["rows"][i]
        print(f"  {ts}: close={row['close']}, volume={row['volume']}")

asyncio.run(main())
```

```bash
uv run main.py
```

---

[← Back to category 03](README.en.md)
