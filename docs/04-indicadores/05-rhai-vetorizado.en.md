# Vectorized Rhai

> `materializar_indicador` — express indicator calculations in a single line of vectorized Rhai.

The `ct-mcp-server` embeds the [Rhai](https://rhai.rs/) language as its scripting engine. The `materializar_indicador` tool lets you write a **single expression** that chains indicators over the input series — no multi-step pipeline needed.

---

## The `Serie` type

Series data enters as variables in the Rhai scope:

| Variable | Type | What it is |
|---|---|---|
| `open` | `Serie` | Open column |
| `high` | `Serie` | High column |
| `low` | `Serie` | Low column |
| `close` | `Serie` | Close column |
| `volume` | `Serie` | Volume column |

### `Serie` operators

```rhai
close + open          // series + series
close - open          // subtraction
close * volume        // multiplication
close / volume        // division
close[0]              // scalar value at current bar
close[1]              // scalar value at previous bar
avg(close)            // mean of entire series
sum(close)            // sum
highest(close, 20)    // highest of last 20
lowest(close, 20)     // lowest of last 20
```

### Utility functions

| Function | What it does |
|---|---|
| `na()` | Not-a-Number |
| `nz(x)` | Null/NaN → 0 |
| `clamp(x, min, max)` | Clamp value |
| `sinal(x)` | Sign: −1, 0, +1 |
| `abs(x)` | Absolute value |

### Indicators as series→series functions

45+ indicators are available as functions that take and return `Serie`:

```rhai
sma(close, 14)                  // SMA 14
ema(close, 9)                   // EMA 9
rsi(close, 14)                 // RSI 14
macd(close)["hist"]             // MACD → get hist column (map)
bollinger(close)["upper"]       // Bollinger → get upper
atr(high, low, close, 14)       // ATR 14
obv(close, volume)             // OBV
stochastic(high, low, close)["k"]   // Stochastic %K
adx(high, low, close)["adx"]        // ADX
```

> Multi-output indicators (MACD, Bollinger, Stochastic, ADX, etc.) return a **map** — access with `["column_name"]`.

---

## Examples

### SMA of RSI

```rhai
sma(rsi(close, 14), 5)
```

### Signal: RSI < 30 → +1, RSI > 70 → −1, else 0

```rhai
let r = rsi(close, 14);
if r < 30.0 { 1.0 } else if r > 70.0 { -1.0 } else { 0.0 }
```

> **Note:** this returns a scalar (last bar). For a full series, use `condicional` in the pipeline or return `#{ "sinal": r.map(|x| if x < 30.0 { 1.0 } else if x > 70.0 { -1.0 } else { 0.0 }) }`.

### Multiple columns (map)

```rhai
#{
  "sma_short": sma(close, 9),
  "sma_long": sma(close, 21),
  "spread": sma(close, 9) - sma(close, 21)
}
```

---

## Tool call

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_rsi_sma",
    "receita": "sma(rsi(close, 14), 5)"
  }
}
```

**Return:**
```json
{
  "uri": "ct://derived/btc_rsi_sma",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "valid_from_ts": 1784068200,
  "value_names": ["sma"]
}
```

### With parameters

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_rsi_signal",
    "receita": "if rsi(close, par[\"p\"]) < 30.0 { 1.0 } else if rsi(close, par[\"p\"]) > 70.0 { -1.0 } else { 0.0 }",
    "parametros": { "p": 14 }
  }
}
```

---

## Pipeline vs Rhai — when to use which?

| Situation | Use |
|---|---|
| 1 indicator | Direct tool (`sma`, `rsi`) |
| Linear expression (1 line) | **Rhai** (`materializar_indicador`) |
| 4+ steps with branches | **Pipeline** (`montar_pipeline_indicadores`) |
| Cross-asset | `compor_serie` + Rhai or pipeline compose |

> **Rule:** Rhai is linear (one expression). Pipeline supports trees (multiple branches converging).

---

> Next: [Recipe cookbook](./07-cookbook.en.md) · [Custom indicators](./09-indicadores-custom.en.md)
