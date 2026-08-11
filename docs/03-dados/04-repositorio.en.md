# 04 · Local Repository and Cache

Every series ingested by CT Lab is stored in a **local repository** — an on-disk cache managed automatically. This document explains how to inspect, manage, and understand cache limits.

---

## Why a Cache Exists

CT Lab uses a local cache for three reasons:

1. **Performance**: Indicators and pipelines read data repeatedly. The cache avoids re-downloading on every access.
2. **Reproducibility**: A cached series is an immutable *snapshot* (until removed). Quantitative methods depend on deterministic data.
3. **API cost**: Providers rate-limit requests. The cache reduces external calls.

---

## Repository Management Tools

| Tool | Description |
|------|-------------|
| `listar_series` | List all cached series with metadata |
| `info_serie` | Details of a specific series by URI |
| `remover_serie` | Remove a series from cache |

### `listar_series` — List Everything

#### AI Chat Prompt

> "List all series I have in the cache."

#### Tool Call

```json
{
  "tool": "listar_series",
  "arguments": {}
}
```

#### Expected Return

```json
{
  "series": [
    {
      "uri": "ct://series/binance/BTCUSDT/15m",
      "kind": "raw",
      "row_count": 1000,
      "first_ts": "2026-08-10T18:00:00Z",
      "last_ts": "2026-08-11T15:00:00Z",
      "last_accessed_at": "2026-08-11T15:05:00Z"
    },
    {
      "uri": "ct://series/yahoo/AAPL/1d",
      "kind": "raw",
      "row_count": 2000,
      "first_ts": "2020-01-01T00:00:00Z",
      "last_ts": "2026-08-11T00:00:00Z",
      "last_accessed_at": "2026-08-11T14:30:00Z"
    },
    {
      "uri": "ct://derived/btc_eth",
      "kind": "derived",
      "row_count": 950,
      "first_ts": "2026-08-10T18:00:00Z",
      "last_ts": "2026-08-11T15:00:00Z",
      "last_accessed_at": "2026-08-11T15:10:00Z"
    }
  ],
  "cache_limit": 100
}
```

The `cache_limit` field shows the maximum series count for your plan (1 or 100).

---

### `info_serie` — Series Details

#### AI Chat Prompt

> "Give me detailed info about the BTCUSDT 15m series."

#### Tool Call

```json
{
  "tool": "info_serie",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m"
  }
}
```

#### Expected Return

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "kind": "raw",
  "columns": ["open", "high", "low", "close", "volume"],
  "row_count": 1000,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "last_accessed_at": "2026-08-11T15:05:00Z",
  "source_uris": []
}
```

#### Return Fields

| Field | Description |
|-------|-------------|
| `uri` | Canonical series identifier |
| `kind` | `"raw"` (ingested) or `"derived"` (composed) |
| `columns` | Available columns for reading |
| `row_count` | Number of candles |
| `first_ts` / `last_ts` | Temporal bounds |
| `last_accessed_at` | Last read (used for LRU eviction) |
| `source_uris` | Source URIs (empty for raw; list for derived) |

> **Tip:** `last_accessed_at` updates on every read (including via `tail`/`head`/`sample` resources). This controls LRU eviction.

---

### `remover_serie` — Remove from Cache

#### AI Chat Prompt

> "Remove the daily AAPL series from the cache."

#### Tool Call

```json
{
  "tool": "remover_serie",
  "arguments": {
    "uri": "ct://series/yahoo/AAPL/1d"
  }
}
```

#### Expected Return

```json
{
  "removido": true
}
```

> **Note:** Removing from cache **does not** delete data from the provider. You can re-ingest the series at any time with `buscar_serie`.

---

## Cache Limits by Plan

| Plan | Series Limit | Chunked Backfill |
|------|:-:|:-:|
| **Free** | 1 series | ❌ |
| **Premium** | 100 series | ✅ |

When the limit is reached, new ingestions automatically make room (LRU eviction for OHLCV).

---

## LRU Eviction (Least Recently Used)

For OHLCV series (stored in **SQLite**), CT Lab uses LRU eviction:

- When a new series is ingested and the cache is full, the series with the oldest `last_accessed_at` is **silently removed**.
- The removed series appears in the `evicted_series` field of `IngestResult`.
- Since OHLCV data is **re-downloadable** from the provider, eviction is non-destructive — you can re-fetch the series when needed.

### Example Return with Eviction

```json
{
  "uri": "ct://series/binance/SOLUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": ["ct://series/yahoo/AAPL/1d"]
}
```

The AAPL series was evicted to make room.

---

## EventStore: Trades and Book (Irrecoverable)

Real-time **trades** (`trades_1s`) and **order book** (`book_1s`) data are stored in a separate **EventStore** with different behavior:

| Aspect | SQLite (OHLCV) | EventStore (Trades/Book) |
|--------|----------------|---------------------------|
| Data | Re-downloadable | **Irrecoverable** — live data |
| Eviction | Silent LRU | **Rejects** new collection at cap |
| Strategy | Replaces old series | Protects existing data |

If the EventStore reaches its cap, new collections are **rejected** rather than overwriting existing data. This protects historical trades/book data that cannot be re-obtained.

---

## Why the Cap Exists

The series cap serves three purposes:

1. **Memory and disk**: 1m OHLCV series can have millions of rows. Limiting the cache prevents excessive resource usage.
2. **Multi-tenant fairness**: In shared environments, the cap ensures equitable usage per account.
3. **Encourages discipline**: The cap encourages users to keep only relevant series (using `remover_serie` to clean up).

---

## Recommended Management Workflow

```
  [Ingest new series]
          │
          ▼
  listar_series  ──>  audit cache
          │
          ▼
  remover_serie  ──>  clean obsolete series
          │
          ▼
  info_serie  ──>  verify before using in pipelines
```

---

## TypeScript Example

```typescript
// List everything
const repo = await Ct.listarSeries({});
console.log(`Series in cache: ${repo.series.length}/${repo.cache_limit}`);
repo.series.forEach(s => {
  console.log(`  ${s.uri} (${s.row_count} candles) — accessed ${s.last_accessed_at}`);
});

// Specific details
const info = await Ct.infoSerie({
  uri: "ct://series/binance/BTCUSDT/15m",
});
console.log(`Columns: ${info.columns.join(", ")}`);

// Clean up a series
await Ct.removerSerie({
  uri: "ct://series/yahoo/AAPL/1d",
});
console.log("Series removed.");
```

---

[← Back to category 03](README.en.md)
