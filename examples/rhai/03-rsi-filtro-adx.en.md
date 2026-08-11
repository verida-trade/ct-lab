# Recipe 03 — RSI with ADX Filter

> **Level:** Intermediate · **Prerequisites:** [Recipe 01](./01-cruzamento-sma.en.md)

Strategy: buy when RSI < 30 **AND** ADX > 25 (strong trend). Sell when RSI > 70.

## Step 1 — Fetch series

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

## Step 2 — Backtest with inline indicators

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" },
      "adx": { "receita": "adx(high, low, close)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_adx_filter"
  }
}
```

## Variations

- **ADX as directional gate:** only buy if DI+ > DI− AND ADX > 25
- **Tighter RSI:** oversold 25, overbought 75
- **Add ATR for sizing:** `atr(high, low, close, 14)` and adjust lot by volatility
