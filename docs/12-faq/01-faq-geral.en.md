# General FAQ

> Frequently asked questions for those getting started with CT Lab.

## Which AI provider do I need?

OpenAI gpt-4o or Anthropic claude-sonnet-4-20250514 — both have excellent MCP support. Ollama runs locally (offline) but may have limitations with function calling. Google Gemini is also supported.

| Provider | Requires internet | Function calling | Recommended |
|---|---|---|---|
| OpenAI (gpt-4o) | ✅ | ✅ Excellent | ⭐ Beginners |
| Anthropic (claude-sonnet) | ✅ | ✅ Excellent | ⭐ Beginners |
| Google (gemini-2.0-flash) | ✅ | ✅ Good | |
| Ollama (llama3, mistral) | ❌ | ⚠️ Varies by model | Full privacy |

## Can I use it offline?

Partially. Ollama runs 100% locally (no cloud), but you need internet to fetch market data (candles, klines). The series cache is local — once data is cached, backtests and indicators work offline.

## What's the difference between `auto` and `chat`?

| Mode | Behavior | When to use |
|---|---|---|
| `auto` | Agent: decides, calls tools, executes | Analysis, backtest, pipeline |
| `chat` | Direct conversation: only responds, no tool execution | Conceptual questions, brainstorm |

To use ct-mcp-server, **always use `auto`**. In `chat` mode the AI cannot invoke tools.

## How many tools does ct-mcp-server offer?

~120 MCP tools (varies by plan and environment variables):

| Category | Count | Plan |
|---|---|---|
| Public indicators | 36 | Free |
| Proprietary CT indicators | 17 | Premium |
| Data (discovery, ingestion) | ~15 | Free |
| Backtest | 2 | Free |
| ML pipeline | ~20 | Premium |
| Microstructure | ~10 | Free/Premium |
| Libs, filters, prompts | ~10 | Free/Premium |
| Billing, config | ~5 | Free |

> The exact number changes across versions. To see the full list: ask the AI "list all available tools" or use `tools/list` in the MCP protocol.

## What does "1 series" mean in the Free plan?

In the Free plan, the series cache holds **1 series**. This means:

- ✅ Fetch `BTCUSDT 15m` → 1 series in cache
- ❌ Fetch `ETHUSDT 15m` without removing the first → `series limit reached` error
- ✅ Remove the previous series (`remover_serie`) and fetch another

```python
# Free: remove before fetching a new series
remover_serie(uri="ct://series/binance/BTCUSDT/15m")
buscar_serie(provider="binance", symbol="ETHUSDT", timeframe="15m")
```

In Premium: up to 100 simultaneous series, with automatic LRU eviction.

## What's the difference between `snake_case` and `camelCase`?

| Context | Notation | Example |
|---|---|---|
| MCP protocol (direct) | `snake_case` | `buscar_serie`, `ct_backtest` |
| TypeScript SDK | `camelCase` | `buscarSerie`, `ctBacktest` |
| Python scripts | `snake_case` | `buscar_serie` |

The AI and MCP protocol use `snake_case`. The TypeScript SDK (desktop app) converts to `camelCase`. In Python scripts, use `snake_case`.

## How does the `ct://` URI system work?

Everything in CT Lab is referenced by a `ct://` URI:

| URI | What it is |
|---|---|
| `ct://series/binance/BTCUSDT/15m` | Raw OHLCV series |
| `ct://derived/btc_rsi` | Derived series (indicator) |
| `ct://backtest/a1b2c3` | Backtest result |
| `ct://models/my_model` | Trained ML model |
| `ct://license/info` | License status |

> MCP tools **never** return raw data rows — they return a URI + metadata. To read the data, use the URI as a resource (`ct://series/.../tail/100`).

## Do I need to install anything besides CT Lab Desktop?

No. `ct-mcp-server` is bundled with the desktop app — it starts automatically.

For **ML** (Premium), you need `uv` installed:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

For **custom Python** (custom indicators in Python), `uv` is also required. Rhai is embedded — zero dependencies.

## How do I pass parameters to indicators?

Each indicator exposes its parameters in the MCP tool call. Defaults are applied if you omit them:

```python
# RSI with default period (14)
rsi(uri="ct://series/binance/BTCUSDT/15m")
# → ct://derived/rsi_...

# RSI with custom period
rsi(uri="ct://series/binance/BTCUSDT/15m", period=21)
# → ct://derived/rsi_...

# MACD with custom fast/slow/signal
macd(uri="ct://series/binance/BTCUSDT/15m", fast=10, slow=30, signal=5)
```

| Indicator | Parameters | Defaults |
|---|---|---|
| `rsi`, `sma`, `ema`, `bollinger` | `period` | 14, 20, 20, 20 |
| `macd` | `fast`, `slow`, `signal` | 12, 26, 9 |
| `stochastic` | `period`, `smooth` | 14, 3 |
| `ichimoku` | `tenkan`, `kijun`, `senkou` | 9, 26, 52 |
| `psar` | `af_step`, `af_max` | 0.02, 0.2 |
| `adx` | `period` | 14 |

> Check the live catalog for the full list: `ct://indicators/catalog`

## What's the difference between `materializar_indicador` and `montar_pipeline_indicadores`?

| Tool | When to use | Complexity |
|---|---|---|
| `materializar_indicador` | A single Rhai expression (e.g.: `sma(rsi(close, 14), 5)`) | Simple |
| `montar_pipeline_indicadores` | Multiple steps with DAG, declarative ops, composition | Advanced |

```python
# Simple: materializar_indicador
materializar_indicador(
    fonte="ct://series/binance/BTCUSDT/15m",
    name="btc_rsi_sma",
    receita="sma(rsi(close, 14), 5)"
)

# Advanced: pipeline with multiple steps
montar_pipeline_indicadores({
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "sinal_cross",
    "output": "$sinal",
    "steps": [
        { "id": "fast", "op": "sma", "source": "$anchor", "period": 9 },
        { "id": "slow", "op": "sma", "source": "$anchor", "period": 21 },
        { "id": "cruz", "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" }
    ]
})
```

## Does `close[0]` mean "current bar" or "first element"?

It depends on the context:

| Context | `close[0]` | `close` |
|---|---|---|
| **Backtest** (Rhai strategy) | Scalar value of the current bar | Not used directly — always indexed |
| **materializar_indicador** (vectorized Rhai) | First element of the series | Full series (array) |

```rhai
// Backtest: close[0] = current bar, close[1] = previous
if close[0] > close[1] { comprado(1.0) }

// Vectorized: close = full series, sma operates over the entire array
sma(close, 14)
```

## How do I use `comparar` vs `condicional` in the pipeline?

| Op | What it does | Example |
|---|---|---|
| `comparar` | Series × series → 0/1 | Did fast SMA cross above slow SMA? |
| `condicional` | Series vs scalar → 0/1 | Is RSI < 30? |

```json
// comparar: series × series
{ "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" }

// condicional: series vs scalar (RSI < 30)
{ "op": "condicional", "condicao": "$rsi", "coluna_condicao": "rsi", "entao": {"escalar": 1.0}, "senao": {"escalar": 0.0} }
```

> **Attention:** `comparar` is always series×series. For threshold against a scalar (`rsi > 30`), use `condicional`.

## How do I create a custom indicator?

Use the `custom` step in the pipeline. Rhai for simple logic (no dependencies), Python for complex calculations:

```json
// Rhai custom (faster, no dependencies)
{
  "id": "my_signal",
  "op": "custom",
  "script": "let r = rsi(close, par[\"p\"]); if r > 70.0 { -1.0 } else if r < 30.0 { 1.0 } else { 0.0 }",
  "entradas": [{ "alias": "close", "fonte": "$anchor", "coluna": "close" }],
  "parametros": { "p": 14 },
  "coluna_saida": "signal"
}

// Python custom (with numpy, scipy, etc.)
{
  "id": "zscore_py",
  "op": "custom",
  "script": "import numpy as np\ndef calcular(close, par):\n    r = np.array(close)\n    return {\"z\": ((r - r.mean()) / r.std()).tolist()}",
  "entradas": [{ "alias": "close", "fonte": "$anchor", "coluna": "close" }],
  "deps": ["numpy"]
}
```

## How do parameters work in Rhai backtest scripts?

Variables declared with `let` persist across bars. Parameters are accessible via `par["name"]`:

```rhai
// State persists across bars
let entry = 0.0;
let stop = 0.0;

if posicao == 0.0 && ind["rsi"][0] < par["limite"] {
    entry = close[0];
    stop = close[0] * (1.0 - par["stop_pct"]);
    comprado(1.0)
} else if posicao > 0.0 && close[0] < stop {
    zerado()
} else {
    comprado(1.0)
}
```

| Variable | What it is |
|---|---|
| `close[0]`, `high[0]`, `low[0]` | Current bar |
| `close[1]` | Previous bar |
| `posicao` | Current position (0=flat, >0=long) |
| `ind["alias"][0]` | Current indicator value |
| `par["name"]` | Strategy parameter |

> Always use `f64` (1.0) not `int` (1) in Rhai arguments.

> Back to: [README](./README.en.md)
