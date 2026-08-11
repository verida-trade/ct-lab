# 03 · Data Management

Welcome to the CT Lab **Data Management** section. This is the foundation of everything: before you can compute indicators, build pipelines, or train ML models, you need data — knowing where to find it, how to ingest it, how to store it, and how to combine multiple series into a single view.

CT Lab treats OHLCV data (candlesticks) as first-class citizens: every ingested series gets a canonical **URI** (`ct://series/...`), is stored in local cache, and can be read point-by-point by the AI model via resources without bloating the context.

---

## Document Index

| # | Document | What it covers |
|---|----------|-----------------|
| 01 | [Asset Discovery](01-descoberta.en.md) | `top_ativos` (gainers/losers/volume) and `filtrar_ativos` (screener), plus persisting screeners with `salvar_filtro` / `listar_filtros` / `excluir_filtro`. |
| 02 | [OHLCV Ingestion](02-ingestao-ohlcv.en.md) | `buscar_serie` (generic), `buscar_binance`, `buscar_yahoo`, `importar_csv`. The `until` param for scroll-back pagination. `IngestResult` shape. |
| 03 | [Chunked Historical Backfill](03-historico-chunked.en.md) | `buscar_binance_historico` and `buscar_serie_historico` for backfilling 180 days of 1-minute data. **Premium.** |
| 04 | [Local Repository](04-repositorio.en.md) | `listar_series`, `info_serie`, `remover_serie`. Cache limits (1 free / 100 premium), LRU eviction, SQLite vs EventStore differences. |
| 05 | [Series Composition](05-composicao.en.md) | `compor_serie` — inner-join of N series by timestamp. The *anchor* concept. Complete BTC/ETH spread example. |
| 06 | [Data URIs](06-uris-dados.en.md) | Resource templates (`tail`, `head`, `sample`). How the AI model reads data point-by-point via resources, not via tool returns. |
| 07 | [Live Catalogs](07-catalogos.en.md) | `ct://sources/catalog`, `ct://indicators/catalog`, `ct://pipeline/catalog`, `ct://ml/catalog` — always current, never hardcoded. |

---

## Conceptual Flow

```
  Discovery          Ingestion           Repository          Composition
  ┌──────────┐       ┌──────────┐        ┌──────────┐       ┌──────────┐
  │top_ativos│──┐    │buscar_   │──┐     │listar_   │──┐    │compor_   │
  │filtrar_  │  ├──> │serie     │  ├──>  │series    │  ├──> │serie     │──> analysis
  │ativos    │  │    │buscar_   │  │     │info_     │  │    │(anchor + │
  └──────────┘  │    │binance   │  │     │serie     │  │    │ inner-join)│
                │    │buscar_   │  │     │remover_  │  │    └──────────┘
                │    │yahoo     │  │     │serie     │  │
                │    │importar_ │  │     └──────────┘  │
                │    │csv       │  │                   │
                │    └──────────┘  │                   │
                └──────────────────┘───────────────────┘
```

---

## Conventions Used in These Docs

- **MCP tools** appear in `snake_case` (e.g., `buscar_serie`) in JSON/RPC payloads.
- **TypeScript SDK** appears in `camelCase` (e.g., `buscarSerie()`) in code blocks.
- **AI chat prompts** appear as blockquotes (`> `).
- **Expected outputs** show the JSON return shape, sometimes abbreviated with `// ...`.

---

## Prerequisites

1. CT Lab installed (desktop app or CLI).
2. `ct-mcp-server` connected to CT Lab (desktop app or CLI).
3. Accounts configured for providers that require API keys (Binance is free; Yahoo available to all).

---

[← Back to root docs](../README.en.md)
