# Backtest Concepts

> How the `ct-mcp-server` backtest engine works — bar-by-bar cycle, inputs and outputs.

## Overview

`ct_backtest` simulates the execution of a strategy bar-by-bar over an OHLCV series. At each bar, the strategy receives data up to the current moment and decides a position: long, short, or flat.

```
Bar 1 → [OHLCV + indicators] → strategy decides → position
Bar 2 → [OHLCV + indicators] → strategy decides → position
...
Bar N → result: equity curve, metrics, trades
```

---

## Strategy inputs

At each bar, the strategy receives:

| Variable | Type | What it is |
|---|---|---|
| `open[0]` | f64 | Current bar open |
| `high[0]` | f64 | Current bar high |
| `low[0]` | f64 | Current bar low |
| `close[0]` | f64 | Current bar close |
| `volume[0]` | f64 | Current bar volume |
| `close[1]` | f64 | Previous bar close |
| `posicao` | f64 | Current position (0=flat, >0=long, <0=short) |
| `ind["alias"][0]` | f64 | Current indicator value by alias |
| `par["name"]` | f64 | Strategy parameter |

> `[0]` = current bar, `[1]` = previous bar, `[k]` = k bars ago.

## Output: Alvo (target)

The strategy returns a "target" — the desired position:

| Rhai function | Meaning |
|---|---|
| `comprado(1.0)` | Go long with 1.0 size |
| `vendido(1.0)` | Go short with 1.0 size |
| `zerado()` | Close position (flat) |
| `decisao(...)` | Custom order (advanced, via grupo lib) |

> **Important:** always use f64 (1.0) not int (1) in Rhai arguments.

---

## Tool call

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "example"
  }
}
```

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `serie` | string | required | OHLCV series URI |
| `estrategia_script` | string | — | Inline Rhai script |
| `estrategia` | string | — | URI to .rhai/.py file (alternative) |
| `indicadores` | string | — | URI of derived series with indicators |
| `indicadores_receitas` | object | — | Alias → Rhai recipe inline |
| `parametros` | object | — | Parameters exposed as `par["name"]` |
| `capital_inicial` | number | — | Initial capital |
| `fee_pct` | number | 0 | Fee per trade (0.001 = 0.1%) |
| `nome` | string | auto | Backtest name/id |

---

## Indicators in backtest

Two ways to pass indicators to the strategy:

### (a) Derived series URI (pre-materialized pipeline)

```json
{ "indicadores": "ct://derived/my_indicators" }
```

All columns from the derived series become aliases: `ind["column_name"][0]`.

### (b) Inline recipes

```json
{
  "indicadores_receitas": {
    "rsi14": { "receita": "rsi(close, 14)" },
    "sma9": { "receita": "sma(close, 9)" }
  }
}
```

The tool materializes each recipe over the series and injects as `ind["alias"]`. The strategy declares what it needs, without pre-materializing.

---

> Next: [Declarative strategy](./02-estrategia-declarativa.en.md)
