# Providers — Binance Spot vs Futures

> Differences between `binance` (Spot) and `binance_um` (Futures USDⓈ-M).

| Aspect | `binance` (Spot) | `binance_um` (Futures) |
|---|---|---|
| Timestamps | μs from 2025-01-01; ms before | ms |
| Book L2 | No bulk; REST snapshot | `bookDepth/` aggregated ±% |
| WS streams | `@aggTrade`, `@depth@100ms` | `@aggTrade`, `@depth@100ms` |
| Trade backfill | `aggTrades` daily ZIP | `aggTrades` daily ZIP |
| URIs | `ct://series/binance/...` | `ct://series/binance_um/...` |

> Provider appears in the URI — distinct data, distinct URI.

---

> Back to: [README](./README.en.md)
