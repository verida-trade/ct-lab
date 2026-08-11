# The MCP Protocol in CT Lab

> **Folder:** `docs/02-conceitos/05-mcp-protocolo.en.md`  
> **Related reading:** [`03-uris`](./03-uris.en.md) ·
> [`01-visao-geral`](./01-visao-geral.en.md)  
> **Target audience:** developers and integrators

---

## What is MCP?

**MCP** (Model Context Protocol) is an open standard that lets LLMs discover
and invoke tools, read resources, and present guided prompts to the user. In
CT Lab, MCP is the single communication protocol between the AI and the local
backend (`ct-labd`).

The `ct-mcp-server` runs in **stdio mode** (JSON-RPC over stdin/stdout),
bridging the LLM and `ct-labd`.

---

## The 4 MCP Primitives

MCP defines four primitives. In CT Lab, each has a specific role:

```
┌──────────────────────────────────────────────────────────────┐
│                     MCP in CT Lab                             │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────┐ │
│  │   Tools    │  │ Resources  │  │  Prompts   │  │Complet.│ │
│  │ (actions)  │  │  (reads)   │  │ (workflows)│  │(autocomplete)│
│  │ snake_case │  │ ct:// URIs │  │ user invoke│  │ args   │ │
│  └────────────┘  └────────────┘  └────────────┘  └────────┘ │
│       │              │                │                │      │
│       ▼              ▼                ▼                ▼      │
│  Returns URI    Returns data     Returns structured  Argument │
│  + metadata     (actual rows)     plan               suggestions│
└──────────────────────────────────────────────────────────────┘
```

| Primitive | Who invokes | What it returns | Example |
|-----------|-------------|-----------------|---------|
| **Tool** | The AI (model) | URI + metadata | `buscar_serie` → `ct://series/...` |
| **Resource** | The AI (model) reads | Actual data (rows) | `ct://series/.../tail/20` |
| **Prompt** | The **user** invokes | Structured workflow | `backtest`, `saudacao` |
| **Completion** | The UI client | Argument suggestions | Autocomplete `provider` |

---

## 1. Tools

Tools are **actions** the AI invokes to discover, create, or transform
resources. The critical principle: **tools never return raw data rows**. They
return a **URI + metadata**.

### Naming Convention

| Environment | Notation | Example |
|-------------|----------|---------|
| MCP (direct protocol) | `snake_case` | `buscar_serie`, `ct_backtest` |
| TypeScript SDK | `camelCase` | `buscarSerie`, `ctBacktest` |

### Tool Examples

```python
# MCP: the AI invokes buscar_serie (snake_case)
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

```typescript
// TypeScript SDK: camelCase
const uri = await Ct.buscarSerie({
    provider: "binance",
    symbol: "BTCUSDT",
    timeframe: "1h",
});
// → { uri: "ct://series/binance/BTCUSDT/1h", meta: { ... } }
```

### Tool Categories

| Category | Examples | Plan |
|----------|---------|------|
| **Data** | `buscar_serie`, `importar_csv`, `listar_series` | Free |
| **Public indicators** (36) | `sma`, `ema`, `rsi`, `macd`, `bollinger`, … | Free |
| **CT indicators** (17) | `ct_bop`, `ct_tfi`, `ct_bfi`, `ct_obi`, … | Premium |
| **Backtest** | `ct_backtest`, `ct_buscar_backtests` | Free |
| **ML** | `montar_esteira_ml`, `aplicar_modelo`, `otimizar_hiperparametros` | Premium |
| **Microstructure** | `coletar_book`, `coletar_trades`, `consultar_book` | Premium |
| **Libs** | `salvar_lib`, `ler_lib`, `listar_libs` | Premium |
| **Collection** | `criar_tarefa_coleta`, `parar_coleta`, `listar_tarefas` | Free |
| **Survival** | `ct_testar_sobrevivencia` | Premium |
| **Billing** | `comprar_premium`, `cancelar_assinatura` | Free |

---

## 2. Resources

Resources are **data reads**. The AI reads the actual content of a resource
using its URI. In CT Lab, resources follow templates that allow reading slices
of a series without transferring everything.

### Read Templates

| Template | Syntax | Limit |
|----------|--------|-------|
| `tail` | `ct://series/.../<tf>/tail/<n>` | n ≤ 200 |
| `head` | `ct://series/.../<tf>/head/<n>` | n ≤ 200 |
| `sample` | `ct://series/.../<tf>/sample/<n>` | n ≤ 200 |

### Example: Reading Data via Resource

```python
# The AI found the series via a tool, now READS data via resource
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "data = read_resource('ct://series/binance/BTCUSDT/1h/tail/10'); "
    "print(data.timestamps[:3]); "
    "print(data.columns['close'][:3])"
], capture_output=True, text=True)
print(result.stdout)
# [1700000000, 1700003600, 1700007200]
# [67100.5, 67150.2, 67200.8]
```

### Catalogs as Resources

```python
# The indicator catalog is a live resource
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "catalog = read_resource('ct://indicators/catalog'); "
    "print(catalog)"
], capture_output=True, text=True)
print(result.stdout)
# [{ name: 'sma', category: 'public' }, { name: 'rsi', ... }, ...]
```

---

## 3. Prompts (Guided Workflows)

Prompts are **structured workflows** that the **user** invokes — not the AI.
They guide the conversation with instructions, suggested parameters, and
validations.

### Public Prompts (Free)

| Prompt | Purpose |
|--------|---------|
| `saudacao` | Initial greeting and onboarding |
| `comecar` | Quick start guide |
| `backtest` | Structures a backtest request step by step |

### Premium Prompts

| Prompt | Purpose |
|--------|---------|
| `coleta` | Configures data collection tasks |
| `esteira` | Builds a full ML pipeline |
| `fundacao` | Creates a data foundation for a project |
| `regime` | Market regime analysis |

### How the User Invokes a Prompt

```
  User (in chat UI): /backtest
  
  → The 'backtest' prompt is injected into the LLM's context
  → The LLM receives structured instructions:
    "Ask which asset, timeframe, strategy, parameters…"
  → The LLM conducts a guided dialogue with the user
  → At the end, calls ct_backtest with the collected parameters
```

---

## 4. Completions (Autocomplete)

Completions provide **argument suggestions** for prompts. For example, when
the user starts typing the `provider` argument in the `backtest` prompt, the
completion suggests: `binance`, `yahoo`, `csv`.

```
  User types: /backtest provider=bin┄
                                   ▼
  Completion returns: ["binance"]
  
  User selects: binance
```

> Completions run in the UI client and do not send data to the LLM — they are
> purely client-side suggestion queries to `ct-mcp-server`.

---

## What the Model Sees vs. What the User Invokes

| Primitive | Who initiates | Does the LLM see it? |
|-----------|---------------|----------------------|
| Tool | The AI decides to invoke | ✅ Yes — in its reasoning context |
| Resource | The AI decides to read | ✅ Yes — content enters the context |
| Prompt | The user invokes | ✅ Yes — injected as instruction into context |
| Completion | The UI client requests | ❌ No — processed client-side |

### Typical Flow

```
 1. User:        /backtest
 2. Prompt:      Injected into LLM context
 3. LLM asks:    "Which asset?"        →User: BTCUSDT
 4. LLM asks:    "Which timeframe?"    →User: 1h
 5. LLM decides: Invoke tool buscar_serie
 6. Tool:        Returns ct://series/binance/BTCUSDT/1h
 7. LLM decides: Read resource tail/100
 8. Resource:    Returns 100 OHLCV rows
 9. LLM decides: Invoke tool sma(period=20)
10. Tool:        Returns ct://derived/sma_...
11. LLM decides: Invoke tool ct_backtest(...)
12. Tool:        Returns ct://backtest/a1b2c3
13. LLM:         Presents result to user
```

---

## Next Steps

- [`03-uris`](./03-uris.en.md) — all URI patterns
- [`04-series`](./04-series.en.md) — data model
- [`06-free-vs-premium`](./06-free-vs-premium.en.md) — license gating
