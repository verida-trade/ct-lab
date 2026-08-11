# 04 · Repositório Local e Cache

Cada série ingerida pelo CT Lab é armazenada em um **repositório local** — um cache em disco gerenciado automaticamente. Este documento explica como inspecionar, gerenciar e entender os limites do cache.

---

## Por Que Existe um Cache?

O CT Lab usa um cache local por três motivos:

1. **Desempenho**: Indicadores e pipelines leem dados repetidamente. O cache evita re-download a cada acesso.
2. **Reprodutibilidade**: Uma série no cache é um *snapshot* imutável (até ser removida). Métodos quantitativos dependem de dados determinísticos.
3. **Custo de API**: Provedores limitam a taxa de requisições. O cache reduz chamadas externas.

---

## Ferramentas de Gestão do Repositório

| Ferramenta | Descrição |
|------------|-----------|
| `listar_series` | Lista todas as séries no cache com metadados |
| `info_serie` | Detalhes de uma série específica por URI |
| `remover_serie` | Remove uma série do cache |

### `listar_series` — Listar Tudo

#### Prompt de Chat IA

> "Liste todas as séries que tenho no cache."

#### Chamada de Ferramenta

```json
{
  "tool": "listar_series",
  "arguments": {}
}
```

#### Retorno Esperado

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

O campo `cache_limit` indica o limite máximo de séries para seu plano (1 ou 100).

---

### `info_serie` — Detalhes de uma Série

#### Prompt de Chat IA

> "Me dê informações detalhadas sobre a série BTCUSDT 15m."

#### Chamada de Ferramenta

```json
{
  "tool": "info_serie",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m"
  }
}
```

#### Retorno Esperado

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

#### Campos do Retorno

| Campo | Descrição |
|-------|-----------|
| `uri` | Identificador canônico da série |
| `kind` | `"raw"` (ingerida) ou `"derived"` (composta) |
| `columns` | Colunas disponíveis para leitura |
| `row_count` | Número de candles |
| `first_ts` / `last_ts` | Extremidades temporais |
| `last_accessed_at` | Última leitura (usado para LRU eviction) |
| `source_uris` | URIs de origem (vazio para raw; lista para derived) |

> **Dica:** O `last_accessed_at` é atualizado a cada leitura (incluindo via resources `tail`/`head`/`sample`). Isso controla a eviction LRU.

---

### `remover_serie` — Remover do Cache

#### Prompt de Chat IA

> "Remova a série do AAPL diário do cache."

#### Chamada de Ferramenta

```json
{
  "tool": "remover_serie",
  "arguments": {
    "uri": "ct://series/yahoo/AAPL/1d"
  }
}
```

#### Retorno Esperado

```json
{
  "removido": true
}
```

> **Nota:** Remover do cache **não** exclui os dados do provedor. Você pode re-ingerir a série a qualquer momento com `buscar_serie`.

---

## Limites de Cache Por Plano

| Plano | Limite de Séries | Backfill Chunked |
|-------|:-:|:-:|
| **Free** | 1 série | ❌ |
| **Premium** | 100 séries | ✅ |

Quando o limite é atingido, novas ingestões tentam fazer espaço automaticamente (LRU eviction para OHLCV).

---

## Eviction LRU (Least Recently Used)

Para séries OHLCV (armazenadas em **SQLite**), o CT Lab usa eviction LRU:

- Quando uma nova série é ingerida e o cache está cheio, a série com o `last_accessed_at` mais antigo é **removida silenciosamente**.
- A série removida aparece no campo `evicted_series` do `IngestResult`.
- Como os dados OHLCV são **re-downloadáveis** do provedor, a eviction é não-destrutiva — você pode re-baixar a série quando precisar.

### Exemplo de Retorno com Eviction

```json
{
  "uri": "ct://series/binance/SOLUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": ["ct://series/yahoo/AAPL/1d"]
}
```

A série AAPL foi evictada para abrir espaço.

---

## EventStore: Trades e Book (Irrecoverable)

Dados de **trades em tempo real** (`trades_1s`) e **order book** (`book_1s`) são armazenados em um **EventStore** separado, que tem comportamento diferente:

| Aspecto | SQLite (OHLCV) | EventStore (Trades/Book) |
|---------|----------------|---------------------------|
| Dados | Re-downloadáveis | **Irrecoverable** — dados ao vivo |
| Eviction | LRU silenciosa | **Rejeita** nova coleta no cap |
| Estratégia | Substitui série antiga | Protege dados existentes |

Se o EventStore atingir o cap, novas coletas são **rejeitadas** em vez de substituir dados existentes. Isto protege dados históricos de trades/book que não podem ser re-obtidos.

---

## Por Que o Cap Existe?

O cap de séries serve a três propósitos:

1. **Memória e disco**: Séries OHLCV de 1m podem ter milhões de linhas. Limitar o cache previne consumo excessivo de recursos.
2. **Justiça multi-tenant**: Em ambientes compartilhados, o cap garante que cada conta tenha uso equitativo.
3. **Incentivo à disciplina**: O cap encoraja o usuário a manter apenas séries relevantes (usando `remover_serie` para limpar).

---

## Workflow de Gestão Recomendado

```
  [Ingerir novas séries]
          │
          ▼
  listar_series  ──>  auditoria do cache
          │
          ▼
  remover_serie  ──>  limpeza de séries obsoletas
          │
          ▼
  info_serie  ──>  verificação antes de usar em pipelines
```

---

## Exemplo em TypeScript

```typescript
// Listar tudo
const repositorio = await Ct.listarSeries({});
console.log(`Séries no cache: ${repositorio.series.length}/${repositorio.cache_limit}`);
repositorio.series.forEach(s => {
  console.log(`  ${s.uri} (${s.row_count} candles) — acessado em ${s.last_accessed_at}`);
});

// Detalhes específicos
const info = await Ct.infoSerie({
  uri: "ct://series/binance/BTCUSDT/15m",
});
console.log(`Colunas: ${info.columns.join(", ")}`);

// Limpar uma série
await Ct.removerSerie({
  uri: "ct://series/yahoo/AAPL/1d",
});
console.log("Série removida.");
```

---

[← Voltar para a categoria 03](README.pt.md)
