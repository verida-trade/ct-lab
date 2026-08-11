# The `ct://` URI System

> **Folder:** `docs/02-conceitos/03-uris.en.md`  
> **Related reading:** [`04-series`](./04-series.en.md) ·
> [`05-mcp-protocolo`](./05-mcp-protocolo.en.md)  
> **Target audience:** developers and integrators

---

## What are `ct://` URIs?

`ct://` URIs are CT Lab's **universal addressing system**. Every resource — a
data series, a materialized indicator, an ML model, a backtest result, or the
trading doctrine — is identified and accessed by a URI.

The AI does not receive raw data rows when it invokes a tool. It receives a
**URI + metadata**. To actually read the data, the AI uses **resource
templates** such as `tail`, `head`, or `sample`.

---

## URI Patterns

| Pattern | Type | Description |
|---------|------|-------------|
| `ct://series/<provider>/<symbol>/<timeframe>` | Resource | Raw OHLCV series |
| `ct://series/<provider>/<symbol>/<timeframe>/tail/<n>` | Resource template | Last N rows (cap 200) |
| `ct://derived/<name>` | Resource | Derived or synthetic series |
| `ct://derived/<name>/head/<n>` | Resource template | First N rows |
| `ct://models/<name>` | Resource | Trained ML model |
| `ct://backtest/<id>` | Resource | Backtest result |
| `ct://doutrina` | Resource | Trading doctrine |
| `ct://doutrina/<topic>` | Resource | Specific doctrine topic |
| `ct://sources/catalog` | Resource | Live catalog of data sources |
| `ct://indicators/catalog` | Resource | Live catalog of indicators |
| `ct://pipeline/catalog` | Resource | Live catalog of pipeline ops |
| `ct://ml/catalog` | Resource | Live catalog of ML components |
| `ct://gateway` | Resource | WebSocket gateway `{ ws_url, token }` |
| `ct://host/fingerprint` | Resource | Machine fingerprint |
| `ct://license/info` | Resource | License information |

---

## Anatomy of a Series URI

```
ct://series/<provider>/<symbol>/<timeframe>
          │          │           │
          │          │           └── 1m, 5m, 15m, 1h, 4h, 1d, 1w …
          │          └── BTCUSDT, AAPL, EURUSD …
          └── binance, yahoo, csv, …
```

### Examples

| URI | Meaning |
|-----|---------|
| `ct://series/binance/BTCUSDT/1h` | 1-hour candles of BTCUSDT on Binance |
| `ct://series/yahoo/AAPL/1d` | Daily bars of AAPL on Yahoo Finance |
| `ct://series/binance/ETHUSDT/15m/tail/50` | Last 50 15-min candles of ETHUSDT |

---

## Resource Templates

Resource templates let the AI **read slices of data** without receiving the
entire series. The three main templates are:

| Template | Purpose | Limit |
|----------|---------|-------|
| `tail/<n>` | Last N rows | N ≤ 200 |
| `head/<n>` | First N rows | N ≤ 200 |
| `sample/<n>` | Random sample of N rows | N ≤ 200 |

### Full Syntax

```
ct://series/<provider>/<symbol>/<timeframe>/tail/<n>     → last N
ct://series/<provider>/<symbol>/<timeframe>/head/<n>     → first N
ct://derived/<name>/tail/<n>                             → last N (derived)
ct://derived/<name>/head/<n>                             → first N (derived)
```

### Practical Example

```python
# The AI found the series via a tool, now reads the last 20 rows
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "data = read_resource('ct://series/binance/BTCUSDT/1h/tail/20'); "
    "print(data)"
], capture_output=True, text=True)
print(result.stdout)
# Prints the last 20 OHLCV rows
```

---

## Catalog URIs

Catalogs are **live** resources — they reflect the current system state and
change as new indicators or sources are added.

| URI | Content |
|-----|---------|
| `ct://sources/catalog` | List of available providers (binance, yahoo, csv, …) |
| `ct://indicators/catalog` | List of available indicators (SMA, RSI, MACD, …) |
| `ct://pipeline/catalog` | List of supported pipeline operations |
| `ct://ml/catalog` | List of ML components (models, optimizers, …) |

```python
# List all available indicators in the system
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "catalog = read_resource('ct://indicators/catalog'); "
    "print(catalog)"
], capture_output=True, text=True)
print(result.stdout)
# Indicator list: SMA, EMA, RSI, MACD, Bollinger, ...
```

---

## Infrastructure URIs

| URI | Description |
|-----|-------------|
| `ct://gateway` | Returns `{ ws_url, token }` for WebSocket connection |
| `ct://host/fingerprint` | Unique machine identifier |
| `ct://license/info` | License status (Free/Premium), limits, validity |

```python
# Check license status
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

## How to Construct a URI

### Raw Series

```
ct://series/ + <provider> + / + <symbol> + / + <timeframe>
```

Example: `ct://series/binance/BTCUSDT/1h`

### Derived Series

```
ct://derived/ + <name>
```

Example: `ct://derived/my_sma_signal`

### Reading a Slice (Resource Template)

```
<base_uri> + /tail/ + <n>       (n ≤ 200)
<base_uri> + /head/ + <n>       (n ≤ 200)
```

Example: `ct://derived/my_sma_signal/tail/10`

---

## Rules and Invariants

| Rule | Detail |
|------|--------|
| `<provider>` | Lowercase provider name (binance, yahoo, csv) |
| `<symbol>` | Ticker as used by the provider (BTCUSDT, AAPL) |
| `<timeframe>` | One of the supported values (1m, 5m, 15m, 1h, 4h, 1d, 1w) |
| `<name>` (derived) | Identifier assigned when materializing/composing the series |
| `tail/head` N | N must be ≤ 200 |
| `sample` N | N must be ≤ 200 |

---

## Mapping: Tool → URI

| MCP Tool | Returned URI |
|----------|-------------|
| `buscar_serie` | `ct://series/<provider>/<symbol>/<timeframe>` |
| `importar_csv` | `ct://series/csv/<name>/<timeframe>` |
| `sma` | `ct://derived/sma_<params>` |
| `materializar_indicador` | `ct://derived/<custom_name>` |
| `compor_serie` | `ct://derived/<custom_name>` |
| `ct_backtest` | `ct://backtest/<id>` |
| `montar_esteira_ml` | `ct://models/<name>` |

> **Fundamental principle:** tools **write** and return URIs; resources **read**
> data via templates. The AI never receives raw rows from a tool.

---

## Next Steps

- [`04-series`](./04-series.en.md) — series data model
- [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) — tools vs resources
- [`06-free-vs-premium`](./06-free-vs-premium.en.md) — `ct://license/info`
