# 03 · Chunked Historical Backfill (Premium)

> ⚠️ **Premium Feature.** Chunked historical backfill requires a CT Lab premium subscription. To learn more, use `comprar_premium` or check the billing documentation.

Standard ingestion (`buscar_serie`, `buscar_binance`) returns a limited batch of candles per call (typically 500–1000). For **massive backfill** — for example, 180 days of 1-minute data — CT Lab offers specialized tools that split the period into *chunks* and ingest them automatically.

---

## When to Use Chunked Backfill?

| Scenario | Tool |
|----------|------|
| Last 1000 candles of 15m | `buscar_binance` |
| Last 1000 candles of 1d | `buscar_yahoo` |
| **180 days of 1m candles** (≈ 259,200 candles) | `buscar_binance_historico` |
| **5 years of daily stock data** | `buscar_serie_historico` |

Chunked backfill is necessary when:
- The data volume exceeds a single API call limit.
- You need continuous data since an arbitrary date in the past.
- You want to train ML models with extensive history.

---

## `buscar_binance_historico` — Binance Backfill

Performs periodic backfill from Binance for a specific symbol and interval.

### AI Chat Prompt

> "Backfill 180 days of 1-minute candles for BTCUSDT on Binance."

### Tool Call

```json
{
  "tool": "buscar_binance_historico",
  "arguments": {
    "symbol": "BTCUSDT",
    "interval": "1m",
    "days_back": 180
  }
}
```

### Expected Return

```json
{
  "uri": "ct://series/binance/BTCUSDT/1m",
  "row_count": 259200,
  "first_ts": "2026-02-12T00:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "chunks_processados": 260,
  "tempo_segundos": 145.3,
  "evicted_series": []
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|:--------:|-------------|
| `symbol` | `string` | ✅ | Trading pair (e.g., `"BTCUSDT"`) |
| `interval` | `string` | ✅ | Candle interval: `"1m"`, `"5m"`, `"15m"`, `"1h"`, etc. |
| `days_back` | `number` | ✅ | How many days back to backfill |
| `provider` | `string` | ❌ | Override provider (default: `"binance"`) |

### How It Works Internally

1. Computes `start_date = now - days_back`.
2. Splits the interval `[start_date, now]` into *chunks* aligned with the candle interval.
3. For each chunk, calls the Binance API.
4. Concatenates and stores in local cache as a continuous series.
5. Returns the `IngestResult` with metadata.

> The process is transparent to the user — a single call handles all pagination.

---

## `buscar_serie_historico` — Generic Backfill

Generic version that dispatches to any supported provider.

### AI Chat Prompt

> "Backfill 5 years of daily AAPL candles from Yahoo Finance."

### Tool Call

```json
{
  "tool": "buscar_serie_historico",
  "arguments": {
    "provider": "yahoo",
    "symbol": "AAPL",
    "interval": "1d",
    "days_back": 1825
  }
}
```

### Expected Return

```json
{
  "uri": "ct://series/yahoo/AAPL/1d",
  "row_count": 1265,
  "first_ts": "2021-08-11T00:00:00Z",
  "last_ts": "2026-08-11T00:00:00Z",
  "chunks_processados": 13,
  "tempo_segundos": 22.8,
  "evicted_series": []
}
```

### Difference: `buscar_serie_historico` vs `buscar_binance_historico`

| Aspect | `buscar_serie_historico` | `buscar_binance_historico` |
|--------|--------------------------|-----------------------------|
| Provider | Any (via `provider`) | Binance only |
| `provider` parameter | ✅ Required | ❌ Implicit |
| Use case | Multi-provider | Optimized for Binance |

---

## Important Considerations

### Processing Time

Backfilling 180 days of 1m data can **take minutes**. Estimates below:

| Data | Approx. candles | Est. time |
|------|----------------:|----------:|
| 30 days of 1m | 43,200 | ~25s |
| 90 days of 1m | 129,600 | ~75s |
| 180 days of 1m | 259,200 | ~150s |
| 365 days of 1m | 525,600 | ~300s |

### Cache and Eviction

Each backfill consumes cache space. If the cache is full:

- **SQLite (OHLCV)**: Oldest series are **silently evicted by LRU**. You can re-download them later.
- **EventStore (trades/book)**: Collection is **rejected** if the cap is reached. EventStore data is irrecoverable.

> See [04-repository](04-repositorio.en.md) for details on cache limits and eviction.

### Free vs Premium Plans

| Feature | Free | Premium |
|---------|:----:|:-------:|
| Series cache | 1 series | 100 series |
| Standard ingestion (`buscar_binance`) | ✅ | ✅ |
| Chunked backfill (`buscar_binance_historico`) | ❌ | ✅ |

---

## Complete TypeScript Example

```typescript
// Backfill 180 days of BTCUSDT 1m
const result = await Ct.buscarBinanceHistorico({
  symbol: "BTCUSDT",
  interval: "1m",
  daysBack: 180,
});

console.log(`Series: ${result.uri}`);
console.log(`${result.row_count} candles ingested`);
console.log(`Period: ${result.first_ts} → ${result.last_ts}`);
console.log(`${result.chunks_processados} chunks in ${result.tempo_segundos}s`);
```

### Python Example with `uv`

```python
# Install
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # Backfill 90 days of ETHUSDT 5m
    result = await client.buscar_binance_historico(
        symbol="ETHUSDT",
        interval="5m",
        days_back=90,
    )
    print(f"URI: {result['uri']}")
    print(f"Candles: {result['row_count']}")
    print(f"Chunks: {result['chunks_processados']}")
    print(f"Time: {result['tempo_segundos']}s")

asyncio.run(main())
```

```bash
uv run main.py
```

---

## Next Steps

After backfill, you can:
- **Inspect the series**: `info_serie` with the returned URI.
- **Compose with another series**: `compor_serie` for cross-asset analysis.
- **Compute indicators**: Use any indicator pointing to the URI.
- **Materialize an indicator**: `materializar_indicador` to persist results.

---

[← Back to category 03](README.en.md)
