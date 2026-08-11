# 03 · Backfill Histórico Chunked (Premium)

> ⚠️ **Recurso Premium.** O backfill histórico chunked requer uma assinatura premium do CT Lab. Para saber mais, use `comprar_premium` ou consulte a documentação de billing.

A ingestão padrão (`buscar_serie`, `buscar_binance`) retorna um lote limitado de candles por chamada (tipicamente 500–1000). Para **backfill massivo** — por exemplo, 180 dias de dados de 1 minuto — o CT Lab oferece ferramentas especializadas que dividem o período em *chunks* e ingerem automaticamente.

---

## Quando Usar Backfill Chunked?

| Cenário | Ferramenta |
|---------|------------|
| Últimos 1000 candles de 15m | `buscar_binance` |
| Últimos 1000 candles de 1d | `buscar_yahoo` |
| **180 dias de candles de 1m** (≈ 259.200 candles) | `buscar_binance_historico` |
| **5 anos de dados diários de uma ação** | `buscar_serie_historico` |

O backfill chunked é necessário quando:
- O volume de dados excede o limite de uma única chamada de API.
- Você precisa de dados contínuos desde uma data arbitrária no passado.
- Quer treinar modelos ML com histórico extenso.

---

## `buscar_binance_historico` — Backfill da Binance

Faz backfill periódico da Binance para um símbolo e intervalo específicos.

### Prompt de Chat IA

> "Faça backfill de 180 dias de candles de 1 minuto do BTCUSDT na Binance."

### Chamada de Ferramenta

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

### Retorno Esperado

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

### Parâmetros

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|:-----------:|-----------|
| `symbol` | `string` | ✅ | Par de trading (ex.: `"BTCUSDT"`) |
| `interval` | `string` | ✅ | Intervalo do candle: `"1m"`, `"5m"`, `"15m"`, `"1h"`, etc. |
| `days_back` | `number` | ✅ | Quantos dias passados para fazer backfill |
| `provider` | `string` | ❌ | Sobrescreve o provedor (default: `"binance"`) |

### Como Funciona Internamente

1. Calcula `data_inicio = agora - days_back`.
2. Divide o intervalo `[data_inicio, agora]` em *chunks* alinhados com o intervalo do candle.
3. Para cada chunk, chama a API da Binance.
4. Concatena e armazena no cache local como uma série contínua.
5. Retorna o `IngestResult` com metadados.

> O processo é transparente para o usuário — uma única chamada lida com toda a paginação.

---

## `buscar_serie_historico` — Backfill Genérico

Versão genérica que despacha para qualquer provedor suportado.

### Prompt de Chat IA

> "Faça backfill de 5 anos de candles diários da AAPL no Yahoo Finance."

### Chamada de Ferramenta

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

### Retorno Esperado

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

### Diferença entre `buscar_serie_historico` e `buscar_binance_historico`

| Aspecto | `buscar_serie_historico` | `buscar_binance_historico` |
|---------|--------------------------|-----------------------------|
| Provedor | Qualquer (via `provider`) | Binance apenas |
| Parâmetro `provider` | ✅ Obrigatório | ❌ Implícito |
| Caso de uso | Multi-provedor | Otimizado para Binance |

---

## Considerações Importantes

### Tempo de Processamento

Backfill de 180 dias de 1m pode **demorar minutos**. A tabela abaixo dá estimativas:

| Dados | candles approx. | Tempo estimado |
|-------|-----------------:|----------------:|
| 30 dias de 1m | 43.200 | ~25s |
| 90 dias de 1m | 129.600 | ~75s |
| 180 dias de 1m | 259.200 | ~150s |
| 365 dias de 1m | 525.600 | ~300s |

### Cache e Eviction

Cada backfill consome espaço no cache. Se o cache estiver cheio:

- **SQLite (OHLCV)**: Séries mais antigas são **evictadas por LRU** silenciosamente. Você pode re-baixá-las depois.
- **EventStore (trades/book)**: A coleta é **rejeitada** se o cap for atingido. Dados de EventStore são irrecoveráveis.

> Veja [04-repositorio](04-repositorio.pt.md) para detalhes sobre limites de cache e eviction.

### Plano Free vs Premium

| Recurso | Free | Premium |
|---------|:----:|:-------:|
| Cache de séries | 1 série | 100 séries |
| Ingestão padrão (`buscar_binance`) | ✅ | ✅ |
| Backfill chunked (`buscar_binance_historico`) | ❌ | ✅ |

---

## Exemplo Completo em TypeScript

```typescript
// Backfill de 180 dias de BTCUSDT 1m
const result = await Ct.buscarBinanceHistorico({
  symbol: "BTCUSDT",
  interval: "1m",
  daysBack: 180,
});

console.log(`Série: ${result.uri}`);
console.log(`${result.row_count} candles ingeridos`);
console.log(`Período: ${result.first_ts} → ${result.last_ts}`);
console.log(`${result.chunks_processados} chunks em ${result.tempo_segundos}s`);
```

### Exemplo em Python com `uv`

```python
# Instalar
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # Backfill 90 dias de ETHUSDT 5m
    result = await client.buscar_binance_historico(
        symbol="ETHUSDT",
        interval="5m",
        days_back=90,
    )
    print(f"URI: {result['uri']}")
    print(f"Candles: {result['row_count']}")
    print(f"Chunks: {result['chunks_processados']}")
    print(f"Tempo: {result['tempo_segundos']}s")

asyncio.run(main())
```

```bash
uv run main.py
```

---

## Próximos Passos

Após o backfill, você pode:
- **Inspecionar a série**: `info_serie` com a URI retornada.
- **Compor com outra série**: `compor_serie` para análise cross-asset.
- **Calcular indicadores**: Use qualquer indicador apontando para a URI.
- **Materializar um indicador**: `materializar_indicador` para persistir resultados.

---

[← Voltar para a categoria 03](README.pt.md)
