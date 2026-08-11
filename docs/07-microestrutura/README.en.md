# Microstructure (Premium)

> Real-time trades and order book collection (1-second), microstructure indicators and visualization.

## Documents

| # | Document | Covers |
|---|---|---|
| 1 | [Trades collection 1s](./01-coletar-trades.en.md) | `coletar_trades`, bulk backfill, `trades_1s` schema (41 cols) |
| 2 | [Book collection 1s](./02-coletar-book.en.md) | `coletar_book`, local book reconstruction, `book_1s` schema (51 cols) |
| 3 | [L2 data query](./03-consulta.en.md) | `consultar_trades`, `consultar_book` |
| 4 | [Microstructure indicators](./04-indicadores-micro.en.md) | `tfi`, `bfi`, `obi`, `dbi`, `mpo` + `ct_*` variants |
| 5 | [Collector status](./05-status.en.md) | `coletas_ativas`, status resources (subscribable) |
| 6 | [Volume & storage](./06-storage.en.md) | Planning: ~23 GB/symbol/year, Parquet |
| 7 | [WebSocket gateway](./07-gateway.en.md) | `ct://gateway` — realtime visualization |
| 8 | [Collection tasks](./08-tarefas.en.md) | `criar_tarefa_coleta` — orchestrator with window/recurrence |
| 9 | [Providers](./09-providers.en.md) | Binance Spot vs USDⓈ-M Futures |

> Portuguese version: [README.pt.md](./README.pt.md)
