# 06 · URIs de Dados — Deep Dive

Uma das decisões arquiteturais mais importantes do CT Lab é a **separação entre metadados e dados**. As ferramentas MCP (`buscar_serie`, `compor_serie`, etc.) retornam apenas **metadados** — URI, contagem de linhas, extremidades temporais. Os dados ponto-a-ponto são lidos via **recursos (resources)** usando templates de URI.

Isto evita inflar o contexto do modelo de IA com milhares de candles. O modelo lê apenas o que precisa, quando precisa.

---

## A Separação: Tools vs Resources

| Aspecto | Tools (MCP) | Resources (MCP) |
|---------|-------------|-----------------|
| O que retorna | URI + metadados | Dados ponto-a-ponto (linhas) |
| Quando usar | Ingestão, composição, gestão | Leitura de dados para análise |
| Tamanho do retorno | Pequeno (~100 bytes) | Variável (até ~N linhas) |
| Exemplo | `buscar_serie` → `{ uri, row_count, ... }` | `ct://series/.../tail/5` → `{ rows: [...] }` |

### Por Que Isto Importa

Se `buscar_serie` retornasse 1000 candles de OHLCV no JSON, o contexto do modelo IA seria consumido rapidamente. Em vez disso:
1. A tool retorna apenas a URI e metadados (~100 bytes).
2. O modelo decide **quantas linhas** precisa e em **que posição** do conjunto de dados.
3. O modelo lê via resource URI (ex.: `/tail/5` para ver as últimas 5 linhas).

Isto é **essencial para a economia de contexto** em conversas longas.

---

## Templates de URI de Recursos

### Séries Raw (Ingeridas)

| Template | Descrição |
|----------|-----------|
| `ct://series/<provider>/<symbol>/<interval>/tail/<N>` | Últimas N linhas |
| `ct://series/<provider>/<symbol>/<interval>/head/<N>` | Primeiras N linhas |
| `ct://series/<provider>/<symbol>/<interval>/sample/<N>` | Amostra aleatória de N linhas |

### Séries Derivadas (Compostas)

| Template | Descrição |
|----------|-----------|
| `ct://derived/<name>/tail/<N>` | Últimas N linhas da série derivada |
| `ct://derived/<name>/head/<N>` | Primeiras N linhas |
| `ct://derived/<name>/sample/<N>` | Amostra aleatória de N linhas |

### Exemplos de URIs Completas

```
ct://series/binance/BTCUSDT/15m/tail/5       # 5 últimas velas de BTCUSDT 15m
ct://series/binance/BTCUSDT/15m/head/10      # 10 primeiras velas
ct://series/binance/BTCUSDT/15m/sample/20     # 20 linhas aleatórias
ct://series/yahoo/AAPL/1d/tail/3              # 3 últimos candles diários da AAPL
ct://derived/btc_eth_spread/tail/5            # 5 últimas linhas da série composta
```

---

## Resposta JSON de um Resource

Quando o modelo IA lê um resource URI, recebe um JSON com esta forma:

### Prompt de Chat IA

> "Mostre as últimas 5 velas de BTCUSDT no intervalo de 15m."

### Resource URI Lida pelo Modelo

```
ct://series/binance/BTCUSDT/15m/tail/5
```

### Resposta JSON

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m/tail/5",
  "row_count": 5,
  "columns": ["open", "high", "low", "close", "volume"],
  "timestamps": [
    "2026-08-11T14:00:00Z",
    "2026-08-11T14:15:00Z",
    "2026-08-11T14:30:00Z",
    "2026-08-11T14:45:00Z",
    "2026-08-11T15:00:00Z"
  ],
  "rows": [
    { "open": 63980.0, "high": 64100.0, "low": 63950.0, "close": 64000.0, "volume": 125.5 },
    { "open": 64000.0, "high": 64150.0, "low": 63980.0, "close": 64100.0, "volume":  98.3 },
    { "open": 64100.0, "high": 64200.0, "low": 63950.0, "close": 63950.0, "volume": 152.1 },
    { "open": 63950.0, "high": 64100.0, "low": 63920.0, "close": 64050.0, "volume": 110.7 },
    { "open": 64050.0, "high": 64120.0, "low": 64000.0, "close": 64080.0, "volume":  87.4 }
  ]
}
```

### Campos da Resposta

| Campo | Descrição |
|-------|-----------|
| `uri` | A URI completa do resource (incluindo `/tail/5`) |
| `row_count` | Número de linhas retornadas (pode ser < N se a série tiver menos) |
| `columns` | Nomes das colunas disponíveis |
| `timestamps` | Array de timestamps ISO 8601 (eixo X) |
| `rows` | Array de objetos com chave = nome da coluna |

---

## `tail` vs `head` vs `sample`

| Operação | Posição | Usa `last_accessed_at`? |
|----------|---------|:-----------------------:|
| `tail/N` | Fim da série (mais recente) | ✅ Sim (atualiza LRU) |
| `head/N` | Início da série (mais antigo) | ✅ Sim |
| `sample/N` | Amostra aleatória | ✅ Sim |

> Todas as leituras via resources atualizam `last_accessed_at`, afetando a prioridade de eviction LRU. Séries frequentemente lidas são protegidas da eviction.

---

## Séries Derivadas — Mesmo Padrão

Uma série composta via `compor_serie` produz a URI `ct://derived/<name>` e segue exatamente os mesmos templates:

### Resource URI

```
ct://derived/btc_eth_spread/tail/5
```

### Resposta JSON

```json
{
  "uri": "ct://derived/btc_eth_spread/tail/5",
  "row_count": 5,
  "columns": ["btc_close", "eth_close"],
  "timestamps": [
    "2026-08-11T14:00:00Z",
    "2026-08-11T14:15:00Z",
    "2026-08-11T14:30:00Z",
    "2026-08-11T14:45:00Z",
    "2026-08-11T15:00:00Z"
  ],
  "rows": [
    { "btc_close": 64000.0, "eth_close": 3200.0 },
    { "btc_close": 64100.0, "eth_close": 3210.0 },
    { "btc_close": 63950.0, "eth_close": 3185.0 },
    { "btc_close": 64050.0, "eth_close": 3195.0 },
    { "btc_close": 64080.0, "eth_close": 3205.0 }
  ]
}
```

Note como as colunas match o `as_column` definido na composição.

---

## Fluxo de Uso Recomendado

```
1. Tool: buscar_serie          → retorna URI + metadados
2. Resource: ct://.../head/10  → modelo verifica a estrutura
3. Resource: ct://.../tail/10  → modelo vê os dados mais recentes
4. Resource: ct://.../sample/50 → modelo amostra para inspeção estatística
5. Indicador ou pipeline usa a URI para computar
```

O modelo IA decide dinamicamente quantas linhas ler com base na tarefa. Por exemplo, para calcular um RSI de 14 períodos, o modelo pode ler `/tail/20` (14 + margem).

---

## Exemplo em Python com `uv`

```python
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # Passo 1: Ingerir (tool retorna metadados)
    result = await client.buscar_binance(
        symbol="BTCUSDT",
        interval="15m",
    )
    uri = result["uri"]
    print(f"Ingerido: {uri} ({result['row_count']} candles)")

    # Passo 2: Ler últimas 5 linhas via resource
    resource_uri = f"{uri}/tail/5"
    data = await client.read_resource(resource_uri)
    print(f"Colunas: {data['columns']}")
    for i, ts in enumerate(data["timestamps"]):
        row = data["rows"][i]
        print(f"  {ts}: close={row['close']}, volume={row['volume']}")

asyncio.run(main())
```

```bash
uv run main.py
```

---

[← Voltar para a categoria 03](README.pt.md)
