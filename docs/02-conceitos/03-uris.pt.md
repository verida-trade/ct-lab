# O Sistema de URIs `ct://`

> **Pasta:** `docs/02-conceitos/03-uris.pt.md`  
> **Leitura relacionada:** [`04-series`](./04-series.pt.md) ·
> [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md)  
> **Público-alvo:** desenvolvedores e integradores

---

## O que são URIs `ct://`?

As URIs `ct://` são o **sistema de endereçamento universal** do CT Lab. Todo
recurso — uma série de dados, um indicador materializado, um modelo de ML, um
backtest, ou a doutrina de trading — é identificado e acessado por uma URI.

A IA não recebe linhas de dados cruas quando invoca uma ferramenta. Ela recebe
uma **URI + metadados**. Para ler os dados de fato, a IA usa **templates de
recursos** (resource templates) como `tail`, `head`, ou `sample`.

---

## Padrões de URI

| Padrão | Tipo | Descrição |
|--------|------|-----------|
| `ct://series/<provider>/<symbol>/<timeframe>` | Resource | Série OHLCV bruta |
| `ct://series/<provider>/<symbol>/<timeframe>/tail/<n>` | Resource template | Últimas N linhas (cap 200) |
| `ct://derived/<name>` | Resource | Série derivada ou sintética |
| `ct://derived/<name>/head/<n>` | Resource template | Primeiras N linhas |
| `ct://models/<name>` | Resource | Modelo de ML treinado |
| `ct://backtest/<id>` | Resource | Resultado de um backtest |
| `ct://doutrina` | Resource | Doutrina de trading |
| `ct://doutrina/<topic>` | Resource | Tópico específico da doutrina |
| `ct://sources/catalog` | Resource | Catálogo vivo de fontes de dados |
| `ct://indicators/catalog` | Resource | Catálogo vivo de indicadores |
| `ct://pipeline/catalog` | Resource | Catálogo vivo de operações de pipeline |
| `ct://ml/catalog` | Resource | Catálogo vivo de componentes de ML |
| `ct://gateway` | Resource | Gateway WebSocket `{ ws_url, token }` |
| `ct://host/fingerprint` | Resource | Fingerprint da máquina |
| `ct://license/info` | Resource | Informações de licença |

---

## Anatomia de uma URI de Série

```
ct://series/<provider>/<symbol>/<timeframe>
          │          │           │
          │          │           └── 1m, 5m, 15m, 1h, 4h, 1d, 1w …
          │          └── BTCUSDT, AAPL, EURUSD …
          └── binance, yahoo, csv, …
```

### Exemplos

| URI | Significado |
|-----|-------------|
| `ct://series/binance/BTCUSDT/1h` | Velas de 1 hora do BTCUSDT na Binance |
| `ct://series/yahoo/AAPL/1d` | Barras diárias da AAPL no Yahoo Finance |
| `ct://series/binance/ETHUSDT/15m/tail/50` | Últimas 50 velas de 15 min do ETHUSDT |

---

## Templates de Recursos (Resource Templates)

Templates de recursos permitem que a IA **leia fatias de dados** sem receber
a série inteira. Os três templates principais são:

| Template | Função | Limite |
|----------|--------|--------|
| `tail/<n>` | Últimas N linhas | N ≤ 200 |
| `head/<n>` | Primeiras N linhas | N ≤ 200 |
| `sample/<n>` | Amostra aleatória de N linhas | N ≤ 200 |

### Sintaxe completa

```
ct://series/<provider>/<symbol>/<timeframe>/tail/<n>     → últimas N
ct://series/<provider>/<symbol>/<timeframe>/head/<n>     → primeiras N
ct://derived/<name>/tail/<n>                             → últimas N (derivada)
ct://derived/<name>/head/<n>                             → primeiras N (derivada)
```

### Exemplo prático

```python
# A IA encontrou a série via ferramenta, agora lê os últimos 20 registros
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "data = read_resource('ct://series/binance/BTCUSDT/1h/tail/20'); "
    "print(data)"
], capture_output=True, text=True)
print(result.stdout)
# Imprime as 20 últimas linhas OHLCV
```

---

## URIs de Catálogo

Catálogos são recursos **vivos** — refletem o estado atual do sistema e podem
mudar conforme novos indicadores ou fontes são adicionados.

| URI | Conteúdo |
|-----|----------|
| `ct://sources/catalog` | Lista de providers disponíveis (binance, yahoo, csv, …) |
| `ct://indicators/catalog` | Lista de indicadores disponíveis (SMA, RSI, MACD, …) |
| `ct://pipeline/catalog` | Lista de operações de pipeline suportadas |
| `ct://ml/catalog` | Lista de componentes de ML (modelos, otimizadores, …) |

```python
# Lista todos os indicadores disponíveis no sistema
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "catalog = read_resource('ct://indicators/catalog'); "
    "print(catalog)"
], capture_output=True, text=True)
print(result.stdout)
# Lista de indicadores: SMA, EMA, RSI, MACD, Bollinger, ...
```

---

## URIs de Infraestrutura

| URI | Descrição |
|-----|-----------|
| `ct://gateway` | Retorna `{ ws_url, token }` para conexão WebSocket |
| `ct://host/fingerprint` | Identificador único da máquina |
| `ct://license/info` | Status da licença (Free/Premium), limites, validade |

```python
# Verifica o status da licença
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "info = read_resource('ct://license/info'); "
    "print(info)"
], capture_output=True, text=True)
print(result.stdout)
# { plan: 'free', max_cache: 1, premium: false }
```

---

## Como Construir uma URI

### Série bruta (Raw)

```
ct://series/ + <provider> + / + <symbol> + / + <timeframe>
```

Exemplo: `ct://series/binance/BTCUSDT/1h`

### Série derivada (Derived)

```
ct://derived/ + <name>
```

Exemplo: `ct://derived/my_sma_signal`

### Leitura de fatia (Resource template)

```
<uri_base> + /tail/ + <n>       (n ≤ 200)
<uri_base> + /head/ + <n>       (n ≤ 200)
```

Exemplo: `ct://derived/my_sma_signal/tail/10`

---

## Regras e Invariáveis

| Regra | Detalhe |
|-------|---------|
| `<provider>` | Nome do provedor em minúsculas (binance, yahoo, csv) |
| `<symbol>` | Ticker como usado pelo provedor (BTCUSDT, AAPL) |
| `<timeframe>` | Um dos valores suportados (1m, 5m, 15m, 1h, 4h, 1d, 1w) |
| `<name>` (derivada) | Identificador dado ao materializar/compor a série |
| `tail/head` N | N deve ser ≤ 200 |
| `sample` N | N deve ser ≤ 200 |

---

## Mapeamento: Ferramenta → URI

| Ferramenta MCP | URI Retornada |
|-----------------|---------------|
| `buscar_serie` | `ct://series/<provider>/<symbol>/<timeframe>` |
| `importar_csv` | `ct://series/csv/<name>/<timeframe>` |
| `sma` | `ct://derived/sma_<params>` |
| `materializar_indicador` | `ct://derived/<custom_name>` |
| `compor_serie` | `ct://derived/<custom_name>` |
| `ct_backtest` | `ct://backtest/<id>` |
| `montar_esteira_ml` | `ct://models/<name>` |

> **Princípio fundamental:** ferramentas **escrevem** e retornam URIs; recursos
> **leem** dados via templates. A IA nunca recebe linhas cruas de uma ferramenta.

---

## Próximos Passos

- [`04-series`](./04-series.pt.md) — modelo de dados das séries
- [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) — tools vs resources
- [`06-free-vs-premium`](./06-free-vs-premium.pt.md) — `ct://license/info`
