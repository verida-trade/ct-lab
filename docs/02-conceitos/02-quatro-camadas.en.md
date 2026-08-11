# The 4-Layer Architecture

> **Folder:** `docs/02-conceitos/02-quatro-camadas.en.md`  
> **Related reading:** [`01-visao-geral`](./01-visao-geral.en.md) ·
> [`03-uris`](./03-uris.en.md) · [`04-series`](./04-series.en.md)  
> **Target audience:** developers and quants

---

## Overview

CT Lab is structured into **4 vertical layers**, each with distinct
responsibilities. Upper layers depend on lower ones, but never the reverse —
data flows bottom-up, and intent flows top-down.

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1 — INTENTION & DOCTRINE                                   │
│  From user objective → teaches · suggests method · protects       │
│  goal-first prompts · ct://doutrina/* · blueprints                │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 2 — COMPOSITION                                            │
│  Builds SIGNALS and FEATURES                                       │
│  pipelines · Rhai scripts · compose → ct://derived               │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 3 — CONSUMPTION                                             │
│  Strategies (backtest) and features (ML)                            │
│  ct_backtest · montar_esteira_ml · aplicar_modelo                 │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 4 — DATA                                                    │
│  Series: discover · ingest · repository                            │
│  buscar_serie · importar_csv · ct://series/*                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Layer 1 — Intention & Doctrine

### Responsibility

Translating the user's objective into a safe, structured action plan. This is
the topmost layer — it **teaches** the user about method, **suggests** the
appropriate analytical path, and **protects** against dangerous decisions
(overfitting, lookahead bias, etc.).

### Components

| Component | Description |
|------------|-----------|
| **Prompts** | Guided workflows the user invokes (e.g., `saudacao`, `backtest`, `comecar`) |
| **Doctrine** | Trading principles accessible via `ct://doutrina/*` |
| **Blueprints** | Pre-packaged strategy templates |

### Concrete Example

The user says: *"I want to test a moving average crossover on BTCUSDT."*

The Intention layer (via the `backtest` prompt) recognizes the goal, suggests
reasonable parameters (SMA window, backtest period), warns about common pitfalls
(overfitting on in-sample data), and structures the request for the Consumption
layer to execute the backtest.

```python
# The guided prompt structured the request — executed via Python
import subprocess
result = subprocess.run(
    ["uv", "run", "python", "-c",
     "from ct_lab import run_backtest; "
     "run_backtest(strategy='sma_cross', symbol='BTCUSDT', timeframe='1h')"],
    capture_output=True, text=True
)
print(result.stdout)
```

> **Note:** In practice, the AI calls the MCP tool `ct_backtest` directly. The
> Python example above illustrates an equivalent execution via SDK.

---

## Layer 2 — Composition

### Responsibility

Transforming raw data into **signals** and **features**. This layer combines
indicators, mathematical operations, and conditional logic to produce derived
and synthetic series.

### Components

| Component | Description |
|------------|-----------|
| **Pipelines** | Sequences of operations (`montar_pipeline_indicadores`) |
| **Rhai Scripts** | Expression language for inline transformations |
| **Compose** | Combines N series into a synthetic series (`compor_serie`) |
| **Materialize** | Persists an indicator as a derived series (`materializar_indicador`) |

### Concrete Example

```python
# Build a pipeline: SMA(20) + RSI(14) → derived series "momentum_signal"
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import build_pipeline; "
    "build_pipeline("
    "  source='ct://series/binance/BTCUSDT/1h',"
    "  ops=['sma:20', 'rsi:14'],"
    "  name='momentum_signal'"
    ")"
], capture_output=True, text=True)
print(result.stdout)
# → ct://derived/momentum_signal
```

From this point, the series `ct://derived/momentum_signal` is available for
consumption by the backtest engine or ML pipeline.

---

## Layer 3 — Consumption

### Responsibility

Consuming series (raw, derived, or synthetic) for two purposes: **trading
strategies** (backtest) and **ML features** (training/prediction).

### Components

| Component | Description |
|------------|-----------|
| **Backtest Engine** | `ct_backtest` — simulates strategies with entry/exit rules |
| **ML Pipeline** | `montar_esteira_ml`, `otimizar_hiperparametros`, `aplicar_modelo` |
| **Optimization** | Hyperparameter search over series |
| **Survival Test** | `ct_testar_sobrevivencia` (Premium) — statistical robustness |

### Concrete Example

```python
# Run a backtest over the derived series created in Layer 2
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import ct_backtest; "
    "ct_backtest("
    "  series='ct://derived/momentum_signal',"
    "  strategy='sma_cross',"
    "  params={'fast': 20, 'slow': 50}"
    ")"
], capture_output=True, text=True)
print(result.stdout)
# → ct://backtest/a1b2c3
```

The result is a backtest URI readable via `ct://backtest/a1b2c3`.

---

## Layer 4 — Data

### Responsibility

Discovering, ingesting, and storing financial series. This is the foundation —
without data, no layer above can function.

### Components

| Component | Description |
|------------|-----------|
| **Discovery** | `buscar_serie`, `listar_series`, `top_ativos` |
| **Ingestion** | `importar_csv`, `buscar_binance`, `buscar_yahoo`, `criar_tarefa_coleta` |
| **Repository** | Local series cache (`ct://series/*`) |
| **Providers** | Binance, Yahoo Finance, CSV, and more |

### Concrete Example

```python
# Discover and ingest the 1h BTCUSDT series from Binance
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import buscar_serie; "
    "buscar_serie(provider='binance', symbol='BTCUSDT', timeframe='1h')"
], capture_output=True, text=True)
print(result.stdout)
# → ct://series/binance/BTCUSDT/1h
```

The URI `ct://series/binance/BTCUSDT/1h` now exists in the repository and can
be read by all upper layers.

---

## Full Flow: Across All 4 Layers

```
 User: "Test SMA crossover on Bitcoin 1h with custom RSI"
    │
    ▼
 [Layer 1] Prompt 'backtest' → structures action plan
    │
    ▼
 [Layer 4] buscar_serie → ct://series/binance/BTCUSDT/1h
    │                                     │
    ▼                                     ▼
 [Layer 2] sma(20) + rsi(14) → ct://derived/my_signal
    │
    ▼
 [Layer 3] ct_backtest(strategy='sma_cross') → ct://backtest/abc123
    │
    ▼
    AI presents result to user (with charts)
```

---

## Design Principles

| Principle | Application |
|-----------|-------------|
| **Separation of concerns** | Each layer has a clear role |
| **Universal addressing via URI** | Every series, indicator, model, and backtest has a `ct://` URI |
| **Never return raw rows from tools** | Tools return URI + metadata; data read via resources |
| **Unidirectional flow** | Data goes up (Data→Consumption); intent goes down (Intention→Composition) |
| **Transient cache** | Series can be materialized but the repository is ephemeral |

---

## Next Steps

- [`03-uris`](./03-uris.en.md) — how to address any resource
- [`04-series`](./04-series.en.md) — series data model in detail
- [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) — how the AI interacts
