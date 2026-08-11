# Microestrutura (Premium)

> Coleta de trades e order book em tempo real (1 segundo), indicadores de microestrutura e visualização.

## Documentos

| # | Documento | O que cobre |
|---|---|---|
| 1 | [Coleta de trades 1s](./01-coletar-trades.pt.md) | `coletar_trades`, bulk backfill, schema `trades_1s` (41 colunas) |
| 2 | [Coleta de book 1s](./02-coletar-book.pt.md) | `coletar_book`, reconstrução de book local, schema `book_1s` (51 colunas) |
| 3 | [Consulta de dados L2](./03-consulta.pt.md) | `consultar_trades`, `consultar_book` |
| 4 | [Indicadores de microestrutura](./04-indicadores-micro.pt.md) | `tfi`, `bfi`, `obi`, `dbi`, `mpo` + variantes `ct_*` |
| 5 | [Status de coletores](./05-status.pt.md) | `coletas_ativas`, resources de status (subscribable) |
| 6 | [Volume e storage](./06-storage.pt.md) | Planning: ~23 GB/símbolo/ano, Parquet |
| 7 | [Gateway WebSocket](./07-gateway.pt.md) | `ct://gateway` — visualização realtime |
| 8 | [Tarefas de coleta](./08-tarefas.pt.md) | `criar_tarefa_coleta` — orquestrador com janela/recorrência |
| 9 | [Providers](./09-providers.pt.md) | Binance Spot vs Futures USDⓈ-M |

> Versão em inglês: [README.en.md](./README.en.md)
