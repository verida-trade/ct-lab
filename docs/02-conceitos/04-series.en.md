# Series Data Model

> **Folder:** `docs/02-conceitos/04-series.en.md`  
> **Related reading:** [`03-uris`](./03-uris.en.md) ·
> [`02-quatro-camadas`](./02-quatro-camadas.en.md)  
> **Target audience:** developers and quants

---

## What is a Series?

A **series** in CT Lab is the fundamental data unit. All analysis — indicators,
backtests, ML pipelines — operates on series. A series is defined by three
fields:

```
Series {
    timestamps:  Vec<i64>                   // Unix seconds UTC, strictly increasing
    columns:     BTreeMap<String, Vec<f64>>  // names: ^[a-z][a-z0-9_]*$, ≤32 chars
    kind:        SeriesKind                  // Raw | Derived | Synthetic
}
```

---

## The 3 Series Types

| Type | Origin | How Created | URI |
|------|--------|-------------|-----|
| **Raw** | External provider (Binance, Yahoo, CSV) | `buscar_serie`, `importar_csv` | `ct://series/<provider>/<symbol>/<timeframe>` |
| **Derived** | Computed from **1** source series | `compute_*` (sma, rsi, macd, …) or `materializar_indicador` | `ct://derived/<name>` |
| **Synthetic** | Composed from **N** source series | `compor_serie`, `montar_pipeline_indicadores` | `ct://derived/<name>` |

### Diagram

```
 Raw (provider)           Derived (1 source)        Synthetic (N sources)
 ┌──────────────┐         ┌───────────────┐        ┌───────────────────┐
 │ Binance API  │──┐      │  SMA(20) over   │        │ compose(sma_raw,  │
 │ Yahoo API    │  ├────►│  ct://series/   │────►  │  rsi_raw)         │──► ct://derived/
 │ CSV import   │──┘     │  binance/BTC…   │        │ pipeline(sma,rsi) │    <synthetic_name>
 └──────────────┘         │  /1h            │        └───────────────────┘
                          └───────────────┘
```

---

## Raw Series

**Raw** series are OHLCV data ingested directly from a provider. They are the
raw material for all analysis.

### Standard OHLCV Columns

| Column | Description |
|--------|-------------|
| `open` | Opening price |
| `high` | Highest price |
| `low` | Lowest price |
| `close` | Closing price |
| `volume` | Traded volume |

### Creation

```python
# Discover and ingest the raw BTCUSDT series (Binance, 1h)
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

## Derived Series

**Derived** series are computed from **exactly one** source series, using
`compute_*` operations (such as `sma`, `rsi`, `macd`, etc.) or the
`materializar_indicador` function.

### How It Works

```
ct://series/binance/BTCUSDT/1h   ──compute_sma(20)──►   ct://derived/sma_btcusdt_1h_20
     (1 source)                                              (Derived)
```

### Example: SMA

```python
# Compute 20-period SMA over the raw series and materialize it
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

### Fetch vs. Materialize

| Operation | Effect | Persists? |
|-----------|--------|----------|
| `sma(...)` (fetch) | Returns instant value, does not save | ❌ No |
| `materializar_indicador(...)` | Creates a permanent derived series | ✅ Yes |
| `rematerializar_indicador(...)` | Recomputes an existing derived series | ✅ Yes |

---

## Synthetic Series

**Synthetic** series are composed from **N** source series (N ≥ 2), using
`compor_serie` or `montar_pipeline_indicadores`. They allow combining multiple
indicators and series into a single signal.

### How It Works

```
ct://series/binance/BTCUSDT/1h   ──┐
ct://series/binance/ETHUSDT/1h   ──┼──compose(spread)──► ct://derived/btc_eth_spread
ct://derived/btc_sma20           ──┘                     (Synthetic)
```

### Example: Composition

```python
# Compose series into a synthetic BTC-ETH spread series
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

### Example: Multi-Indicator Pipeline

```python
# Pipeline: apply SMA + RSI + MACD in sequence over a series
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

## Data Model Invariants

| Invariant | Description |
|------------|-------------|
| `timestamps` | Unix seconds UTC, **strictly increasing**, no duplicates |
| `columns` (names) | Regex `^[a-z][a-z0-9_]*$`, max 32 characters |
| `columns` (values) | `Vec<f64>` — always floats, even for integers |
| Alignment | All columns have the same number of elements as `timestamps` |
| `kind` | A series never changes type after creation |

### Column Name Validation

```
✅ open, high, low, close, volume           — standard OHLCV
✅ sma_20, rsi_14, macd_signal              — derived indicators
✅ btc_eth_spread, momentum_score           — synthetic series
❌ Open, SMA_20, 1open, btc-eth             — invalid (uppercase, leading digit, hyphen)
```

---

## Series Lifecycle

```
  ┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
  │Discovery │─────►│ Ingestion│─────►│Repository│─────►│Consumption│
  │buscar_  │      │Local cache│     │ct://     │     │backtest  │
  │serie    │      │up to 1/100│     │series/*  │     │ML models │
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

### Removal

```python
# Remove a series from the repository
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import remover_serie; "
    "remover_serie('ct://derived/btc_sma20')"
], capture_output=True, text=True)
print(result.returncode)  # 0 = success
```

---

## Cache and Limits

| Resource | Free | Premium |
|---------|------|---------|
| Series in cache | 1 | 100 |
| Providers | Binance, Yahoo, CSV | + Microstructure (trades_1s, book_1s) |
| Indicator materialization | ✅ | ✅ |
| Synthetic series composition | ✅ | ✅ |
| `salvar_lib` / `ler_lib` | ❌ | ✅ |

> See [`06-free-vs-premium`](./06-free-vs-premium.en.md) for the full comparison.

---

## Next Steps

- [`03-uris`](./03-uris.en.md) — how to address series
- [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) — tools and resources
- [`06-free-vs-premium`](./06-free-vs-premium.en.md) — per-plan limits
