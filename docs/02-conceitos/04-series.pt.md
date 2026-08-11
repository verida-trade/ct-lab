# Modelo de Dados de Séries

> **Pasta:** `docs/02-conceitos/04-series.pt.md`  
> **Leitura relacionada:** [`03-uris`](./03-uris.pt.md) ·
> [`02-quatro-camadas`](./02-quatro-camadas.pt.md)  
> **Público-alvo:** desenvolvedores e quants

---

## O que é uma Série?

Uma **série** no CT Lab é a unidade fundamental de dados. Toda análise —
indicadores, backtests, pipelines de ML — opera sobre séries. Uma série é
definida por três campos:

```
Series {
    timestamps:  Vec<i64>           // Unix seconds UTC, estritamente crescente
    columns:     BTreeMap<String, Vec<f64>>  // nomes: ^[a-z][a-z0-9_]*$, ≤32 chars
    kind:        SeriesKind         // Raw | Derived | Synthetic
}
```

---

## Os 3 Tipos de Série

| Tipo | Origem | Como é Criada | URI |
|------|--------|---------------|-----|
| **Raw** | Provider externo (Binance, Yahoo, CSV) | `buscar_serie`, `importar_csv` | `ct://series/<provider>/<symbol>/<timeframe>` |
| **Derived** | Computada de **1** série fonte | `compute_*` (sma, rsi, macd, …) ou `materializar_indicador` | `ct://derived/<name>` |
| **Synthetic** | Composta de **N** séries fonte | `compor_serie`, `montar_pipeline_indicadores` | `ct://derived/<name>` |

### Diagrama

```
 Raw (provider)           Derived (1 fonte)        Synthetic (N fontes)
 ┌──────────────┐         ┌───────────────┐        ┌───────────────────┐
 │ Binance API  │──┐      │  SMA(20) sobre │        │ compose(sma_raw,  │
 │ Yahoo API    │  ├────►│  ct://series/   │────►  │  rsi_raw)         │──► ct://derived/
 │ CSV import   │──┘     │  binance/BTC…   │        │ pipeline(sma,rsi) │    <synthetic_name>
 └──────────────┘         │  /1h            │        └───────────────────┘
                          └───────────────┘
```

---

## Série Raw (Bruta)

Séries **Raw** são dados OHLCV ingeridos diretamente de um provider. São a
matéria-prima para todas as análises.

### Colunas padrão OHLCV

| Coluna | Descrição |
|--------|-----------|
| `open` | Preço de abertura |
| `high` | Preço máximo |
| `low` | Preço mínimo |
| `close` | Preço de fechamento |
| `volume` | Volume negociado |

### Criação

```python
# Descobre e ingere a série bruta do BTCUSDT (Binance, 1h)
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import buscar_serie; "
    "uri = buscar_serie(provider='binance', symbol='BTCUSDT', timeframe='1h'); "
    "print(uri)"
], capture_output=True, text=True)
print(result.stdout)
# → ct://series/binance/BTCUSDT/1h
```

---

## Série Derived (Derivada)

Séries **Derived** são calculadas a partir de **exatamente uma** série fonte,
usando operações `compute_*` (como `sma`, `rsi`, `macd`, etc.) ou a função
`materializar_indicador`.

### Como funciona

```
ct://series/binance/BTCUSDT/1h   ──compute_sma(20)──►   ct://derived/sma_btcusdt_1h_20
     (1 fonte)                                              (Derived)
```

### Exemplo: SMA

```python
# Calcula SMA de 20 perímetros sobre a série bruta e materializa
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import materializar_indicador; "
    "uri = materializar_indicador("
    "  source='ct://series/binance/BTCUSDT/1h',"
    "  indicator='sma',"
    "  params={'period': 20},"
    "  name='btc_sma20'"
    "); "
    "print(uri)"
], capture_output=True, text=True)
print(result.stdout)
# → ct://derived/btc_sma20
```

### Diferença: compute fetch vs. materializar

| Operação | Efeito | Persiste? |
|----------|--------|-----------|
| `sma(...)` (fetch) | Retorna valor instantâneo, sem salvar | ❌ Não |
| `materializar_indicador(...)` | Cria série derivada permanente | ✅ Sim |
| `rematerializar_indicador(...)` | Recalcula uma série derivada existente | ✅ Sim |

---

## Série Synthetic (Sintética)

Séries **Synthetic** são compostas a partir de **N** séries fonte (N ≥ 2),
usando `compor_serie` ou `montar_pipeline_indicadores`. Permitem combinar
múltiplos indicadores e séries em um único sinal.

### Como funciona

```
ct://series/binance/BTCUSDT/1h   ──┐
ct://series/binance/ETHUSDT/1h   ──┼──compose(spread)──► ct://derived/btc_eth_spread
ct://derived/btc_sma20           ──┘                     (Synthetic)
```

### Exemplo: Composição

```python
# Compõe séries em uma série sintética de spread BTC-ETH
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import composar_serie; "
    "uri = composar_serie("
    "  sources=['ct://series/binance/BTCUSDT/1h',"
    "           'ct://series/binance/ETHUSDT/1h'],"
    "  operation='spread',"
    "  name='btc_eth_spread'"
    "); "
    "print(uri)"
], capture_output=True, text=True)
print(result.stdout)
# → ct://derived/btc_eth_spread
```

### Exemplo: Pipeline de múltiplos indicadores

```python
# Pipeline: aplica SMA + RSI + MACD em sequência sobre uma série
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import montar_pipeline_indicadores; "
    "uri = montar_pipeline_indicadores("
    "  source='ct://series/binance/BTCUSDT/1h',"
    "  ops=['sma:20', 'rsi:14', 'macd:12,26,9'],"
    "  name='btc_momentum'"
    "); "
    "print(uri)"
], capture_output=True, text=True)
print(result.stdout)
# → ct://derived/btc_momentum
```

---

## Invariáveis do Modelo de Dados

| Invariável | Descrição |
|------------|-----------|
| `timestamps` | Unix seconds UTC, **estritamente crescente**, sem duplicatas |
| `columns` (nomes) | Regex `^[a-z][a-z0-9_]*$`, máximo 32 caracteres |
| `columns` (valores) | `Vec<f64>` — sempre floats, mesmo para inteiros |
| Alinhamento | Todas as colunas têm o mesmo número de elementos que `timestamps` |
| `kind` | Uma série nunca muda de tipo após criação |

### Validação de nomes de coluna

```
✅ open, high, low, close, volume           — padrão OHLCV
✅ sma_20, rsi_14, macd_signal              — indicadores derivados
✅ btc_eth_spread, momentum_score           — séries sintéticas
❌ Open, SMA_20, 1open, btc-eth             — inválidos (maiúscula, dígito inicial, hífen)
```

---

## Ciclo de Vida de uma Série

```
  ┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
  │Discovery │─────►│ Ingestion│─────►│Repository│─────►│Consumption│
  │buscar_  │      │Cache local│     │ct://     │     │backtest  │
  │serie    │      │até 1/100 │      │series/*  │     │ML models │
  └─────────┘      └──────────┘      └──────────┘      └──────────┘
                                         │
                                    ┌─────┴──────┐
                                    ▼            ▼
                              ┌──────────┐  ┌──────────┐
                              │ Derived  │  │ Synthetic│
                              │compute_* │  │compose   │
                              └──────────┘  │pipeline  │
                                            └──────────┘
```

### Remoção

```python
# Remove uma série do repositório
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import remover_serie; "
    "remover_serie('ct://derived/btc_sma20')"
], capture_output=True, text=True)
print(result.returncode)  # 0 = sucesso
```

---

## Cache e Limites

| Recurso | Free | Premium |
|---------|------|---------|
| Séries em cache | 1 | 100 |
| Providers | Binance, Yahoo, CSV | + Microestrutura (trades_1s, book_1s) |
| Materialização de indicadores | ✅ | ✅ |
| Composição de séries sintéticas | ✅ | ✅ |
| `salvar_lib` / `ler_lib` | ❌ | ✅ |

> Veja [`06-free-vs-premium`](./06-free-vs-premium.pt.md) para o comparativo
> completo.

---

## Próximos Passos

- [`03-uris`](./03-uris.pt.md) — como endereçar séries
- [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) — tools e resources
- [`06-free-vs-premium`](./06-free-vs-premium.pt.md) — limites por plano
