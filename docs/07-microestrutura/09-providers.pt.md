# Providers — Binance Spot vs Futures

> Diferenças entre `binance` (Spot) e `binance_um` (Futures USDⓈ-M).

| Aspecto | `binance` (Spot) | `binance_um` (Futures) |
|---|---|---|
| Timestamps | μs a partir de 2025-01-01; ms antes | ms |
| Book L2 | Sem bulk; snapshot REST | `bookDepth/` agregado ±% |
| Streams WS | `@aggTrade`, `@depth@100ms` | `@aggTrade`, `@depth@100ms` |
| Backfill trades | `aggTrades` daily ZIP | `aggTrades` daily ZIP |
| URIs | `ct://series/binance/...` | `ct://series/binance_um/...` |

> O provider aparece na URI — dado distinto, URI distinta.

---

> Voltar para: [README](./README.pt.md)
