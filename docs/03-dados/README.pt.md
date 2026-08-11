# 03 · Gestão de Dados

Bem-vindo à seção de **Gestão de Dados** do CT Lab. Esta é a base de tudo: antes de calcular indicadores, montar pipelines ou treinar modelos ML, você precisa ter dados — sabendo onde buscá-los, como ingerir, como armazenar, e como combinar múltiplas séries em uma única visão.

O CT Lab trata dados OHLCV (candlesticks) como cidadãos de primeira classe: cada série ingerida recebe um **URI** canônico (`ct://series/...`), é armazenada em cache local, e pode ser lida ponto-a-ponto pelo modelo de IA via recursos (resources) sem inflar o contexto.

---

## Sumário de Documentos

| # | Documento | O que cobre |
|---|-----------|-------------|
| 01 | [Descoberta de Ativos](01-descoberta.pt.md) | `top_ativos` (ganhadores/perdedores/volume) e `filtrar_ativos` (screener), além de persistir filtros com `salvar_filtro` / `listar_filtros` / `excluir_filtro`. |
| 02 | [Ingestão OHLCV](02-ingestao-ohlcv.pt.md) | `buscar_serie` (genérico), `buscar_binance`, `buscar_yahoo`, `importar_csv`. Parâmetro `until` para paginação scroll-back. Formato `IngestResult`. |
| 03 | [Backfill Histórico (Chunked)](03-historico-chunked.pt.md) | `buscar_binance_historico` e `buscar_serie_historico` para backfill de 180 dias de dados de 1 minuto. **Premium.** |
| 04 | [Repositório Local](04-repositorio.pt.md) | `listar_series`, `info_serie`, `remover_serie`. Limites de cache (1 free / 100 premium), eviction LRU, diferença entre SQLite e EventStore. |
| 05 | [Composição de Séries](05-composicao.pt.md) | `compor_serie` — inner-join de N séries por timestamp. Conceito de *anchor*. Exemplo completo de spread BTC/ETH. |
| 06 | [URIs de Dados](06-uris-dados.pt.md) | Templates de recursos (`tail`, `head`, `sample`). Como o modelo IA lê dados ponto-a-ponto via resources, não via retorno de tools. |
| 07 | [Catálogos ao Vivo](07-catalogos.pt.md) | `ct://sources/catalog`, `ct://indicators/catalog`, `ct://pipeline/catalog`, `ct://ml/catalog` — sempre atuais, nunca hardcoded. |

---

## Fluxo Conceitual

```
  Descoberta          Ingestão            Repositório         Composição
  ┌──────────┐       ┌──────────┐        ┌──────────┐       ┌──────────┐
  │top_ativos│──┐    │buscar_   │──┐     │listar_   │──┐    │compor_   │
  │filtrar_  │  ├──> │serie     │  ├──>  │series    │  ├──> │serie     │──> análise
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

## Convenções Usadas Nestes Docs

- **Ferramentas MCP** aparecem no formato `snake_case` (ex.: `buscar_serie`) no JSON/RPC.
- **SDK TypeScript** aparece em `camelCase` (ex.: `buscarSerie()`) em blocos de código.
- **Prompts de chat IA** aparecem em citações (`> `).
- **Saídas esperadas** mostram a forma (shape) do JSON retornado, às vezes abreviada com `// ...`.

---

## Pré-requisitos

1. CT Lab instalado (desktop app ou CLI).
2. `ct-mcp-server` conectado ao CT Lab (app desktop ou CLI).
3. Contas configuradas para provedores que exigem API (Binance é livre; Yahoo disponível para todos).

---

[← Voltar para a documentação raiz](../README.pt.md)
