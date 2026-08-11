# Receita 03 — RSI com Filtro ADX

> **Nível:** Intermediário · **Pré-requisitos:** [Receita 01](./01-cruzamento-sma.pt.md)

Estratégia: comprar quando RSI < 30 **E** ADX > 25 (tendência forte). Vender quando RSI > 70.

## Passo 1 — Buscar série

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

## Passo 2 — Backtest com indicadores inline

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

## Variações

- **ADX como gate direcional:** só compra se DI+ > DI− E ADX > 25
- **RSI mais apertado:** oversold 25, overbought 75
- **Adicionar ATR para sizing:** `atr(high, low, close, 14)` e ajustar lote por volatilidade
