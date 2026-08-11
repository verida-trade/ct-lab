# Declarative Strategy (Inline)

> The simplest backtest form: an inline Rhai script passed in `estrategia_script`.

No external files needed — the entire strategy fits in a string.

---

## Example 1: Moving average crossover

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma9\"][0] > ind[\"sma21\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma9": { "receita": "sma(close, 9)" },
      "sma21": { "receita": "sma(close, 21)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "sma_cross"
  }
}
```

## Example 2: RSI oversold/overbought

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_mean_reversion"
  }
}
```

## Strategy via file (URI)

Instead of inline, reference a file:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia": "file:///path/to/my_strategy.rhai",
    "estrategia_hash": "sha256hex...",
    "capital_inicial": 1000
  }
}
```

- `estrategia_hash` is optional to pin script integrity.

---

> Next: [Rhai strategy (advanced)](./03-estrategia-rhai.en.md)
