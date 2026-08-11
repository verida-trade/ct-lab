# 02 · Ingestão OHLCV

A ingestão é o ato de buscar dados de candlesticks (Open-High-Low-Close-Volume) de um provedor externo e armazená-los no cache local do CT Lab. Cada série ingerida recebe um **URI canônico** que pode ser referenciado por indicadores, pipelines e modelos ML.

O CT Lab oferece quatro ferramentas de ingestão:

| Ferramenta | Descrição |
|------------|-----------|
| `buscar_serie` | Genérica — despacha para o provedor correto com base no parâmetro `provider`. |
| `buscar_binance` | Atalho para Binance (equivale a `buscar_serie` com `provider: "binance"`). |
| `buscar_yahoo` | Atalho para Yahoo Finance. |
| `importar_csv` | Importa dados de um arquivo CSV customizado. |

---

## O Formato `IngestResult`

Todas as ferramentas de ingestão retornam um objeto no mesmo formato — chamado `IngestResult`:

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-11T10:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": []
}
```

| Campo | Descrição |
|-------|-----------|
| `uri` | URI canônica da série no repositório local |
| `row_count` | Número de candles ingeridos |
| `first_ts` | Timestamp do primeiro candle |
| `last_ts` | Timestamp do último candle |
| `evicted_series` | Lista de URIs removidas por LRU eviction (cache cheio) |

> **Nota:** O retorno contém apenas metadados. Os dados ponto-a-ponto são lidos via **recursos (resources)** usando URIs como `ct://series/binance/BTCUSDT/15m/tail/10`. Veja [06-uris-dados](06-uris-dados.pt.md) para detalhes.

---

## `buscar_serie` — Ingestão Genérica

A ferramenta genérica despacha para o provedor correto com base no `provider`.

### Prompt de Chat IA

> "Busque 1000 candles de BTCUSDT em intervalo de 15 minutos da Binance."

### Chamada de Ferramenta

```json
{
  "tool": "buscar_serie",
  "arguments": {
    "provider": "binance",
    "symbol": "BTCUSDT",
    "interval": "15m"
  }
}
```

### Retorno Esperado

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": []
}
```

### Parâmetros

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|:-----------:|-----------|
| `provider` | `string` | ✅ | `"binance"`, `"yahoo"`, etc. |
| `symbol` | `string` | ✅ | Par de trading (ex.: `"BTCUSDT"`) ou ticker (ex.: `"AAPL"`) |
| `interval` | `string` | ✅ | Intervalo do candle: `"1m"`, `"5m"`, `"15m"`, `"1h"`, `"4h"`, `"1d"`, etc. |
| `limit` | `number` | ❌ | Número máximo de candles (default varia por provedor) |
| `until` | `string` | ❌ | Timestamp ISO 8601 — busca candles **anteriores** a esta data |

---

## `buscar_binance` — Atalho para Binance

Equivale a `buscar_serie` com `provider` fixado em `"binance"`. Menos um parâmetro para digitar.

### Prompt de Chat IA

> "Baixe candles de BTCUSDT 15m da Binance."

### Chamada de Ferramenta

```json
{
  "tool": "buscar_binance",
  "arguments": {
    "symbol": "BTCUSDT",
    "interval": "15m"
  }
}
```

### Retorno Esperado

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "evicted_series": []
}
```

---

## `buscar_yahoo` — Atalho para Yahoo Finance

Para ações, ETFs e outros instrumentos tradicionais.

### Prompt de Chat IA

> "Busque candles diários da AAPL no Yahoo Finance."

### Chamada de Ferramenta

```json
{
  "tool": "buscar_yahoo",
  "arguments": {
    "symbol": "AAPL",
    "interval": "1d"
  }
}
```

### Retorno Esperado

```json
{
  "uri": "ct://series/yahoo/AAPL/1d",
  "row_count": 1000,
  "first_ts": "2022-09-01T00:00:00Z",
  "last_ts": "2026-08-11T00:00:00Z",
  "evicted_series": []
}
```

---

## Parâmetro `until` — Paginação Scroll-Back

O parâmetro `until` permite buscar candles **anteriores a uma data específica**. Isso é essencial para paginar grandes volumes de histórico sem repetir dados.

### Como Funciona

1. Primeira chamada: sem `until` → retorna os candles mais recentes.
2. Use `first_ts` do retorno como `until` na próxima chamada.
3. Repita até atingir a profundidade desejada.

### Prompt de Chat IA

> "Busque candles de BTCUSDT 15m até 1º de agosto de 2026."

### Chamada de Ferramenta

```json
{
  "tool": "buscar_binance",
  "arguments": {
    "symbol": "BTCUSDT",
    "interval": "15m",
    "until": "2026-08-01T00:00:00Z"
  }
}
```

### Retorno Esperado

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": "2026-07-31T10:00:00Z",
  "last_ts": "2026-08-01T00:00:00Z",
  "evicted_series": []
}
```

### Padrão de Paginação Completo

```typescript
// Paginação scroll-back: buscar 5000 candles em lotes de 1000
let allFirstTs: string | undefined;
const batches = [];

for (let i = 0; i < 5; i++) {
  const result = await Ct.buscarBinance({
    symbol: "BTCUSDT",
    interval: "15m",
    until: allFirstTs,
  });
  batches.push(result);
  // Use first_ts como cursor para o próximo lote
  // (Na prática, leia first_ts da série armazenada)
  if (result.row_count < 1000) break; // não há mais dados
}
```

---

## `importar_csv` — Importação de CSV Customizado

Traga seus próprios dados. Útil para ativos exóticos, dados sintéticos, ou planilhas personalizadas.

### Prompt de Chat IA

> "Importe o arquivo ~/data/my_custom_data.csv como série com símbolo CUSTOM1 no intervalo 1h."

### Chamada de Ferramenta

```json
{
  "tool": "importar_csv",
  "arguments": {
    "path": "~/data/my_custom_data.csv",
    "symbol": "CUSTOM1",
    "interval": "1h",
    "provider": "custom"
  }
}
```

### Formato Esperado do CSV

O arquivo deve conter colunas de timestamp e OHLCV:

```csv
timestamp,open,high,low,close,volume
2026-08-11T10:00:00Z,64000.0,64100.0,63950.0,64080.0,125.5
2026-08-11T11:00:00Z,64080.0,64200.0,64000.0,64150.0,98.3
2026-08-11T12:00:00Z,64150.0,64300.0,64100.0,64250.0,152.1
```

### Retorno Esperado

```json
{
  "uri": "ct://series/custom/CUSTOM1/1h",
  "row_count": 3,
  "first_ts": "2026-08-11T10:00:00Z",
  "last_ts": "2026-08-11T12:00:00Z",
  "evicted_series": []
}
```

---

## Exemplo em Python com `uv`

```python
# Instalar dependências
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # BTCUSDT 15m da Binance
    btc = await client.buscar_binance(
        symbol="BTCUSDT",
        interval="15m",
    )
    print(f"BTC: {btc['row_count']} candles"
          f" ({btc['first_ts']} → {btc['last_ts']})")
    print(f"URI: {btc['uri']}")

    # AAPL 1d do Yahoo
    aapl = await client.buscar_yahoo(
        symbol="AAPL",
        interval="1d",
    )
    print(f"AAPL: {aapl['row_count']} candles")
    print(f"URI: {aapl['uri']}")

    # Paginação scroll-back
    older = await client.buscar_binance(
        symbol="BTCUSDT",
        interval="15m",
        until=btc["first_ts"],
    )
    print(f"Lote anterior: {older['row_count']} candles")

asyncio.run(main())
```

```bash
# Executar
uv run main.py
```

---

## Resumo das URIs por Provedor

| Provedor | Padrão de URI |
|----------|---------------|
| Binance | `ct://series/binance/<symbol>/<interval>` |
| Yahoo | `ct://series/yahoo/<symbol>/<interval>` |
| Custom (CSV) | `ct://series/custom/<symbol>/<interval>` |

> A URI é a identidade da série no repositório. Todas as operações subsequentes (indicadores, composição, ML) referenciam a série por esta URI.

---

[← Voltar para a categoria 03](README.pt.md)
