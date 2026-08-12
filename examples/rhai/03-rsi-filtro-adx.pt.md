# Receita 03 — RSI com Filtro ADX

> **Nível:** Intermediário · **Pré-requisitos:** [Receita 01](./01-cruzamento-sma.pt.md)

Comprar quando RSI < 30 **E** ADX > 25 (tendência forte). Vender quando RSI > 70. O filtro ADX elimina sinais de reversão em mercado lateral.

> ⚠️ **ADX retorna um mapa** (`adx`, `plus_di`, `minus_di`), não uma série única. Por isso **não funciona** como `indicadores_receitas` — é preciso materializar via pipeline com `compose`.

---

## Passo 1 — Buscar série

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

## Passo 2 — Materializar ADX + RSI via pipeline

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_adx_rsi",
    "output": "$concat",
    "steps": [
      { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
      { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
      {
        "id": "concat",
        "op": "compose",
        "columns": [
          { "source": "$adx", "source_column": "adx", "as_column": "adx" },
          { "source": "$rsi", "source_column": "rsi", "as_column": "rsi" }
        ]
      }
    ]
  }
}
```

A série `ct://derived/btc_adx_rsi` contém `adx` e `rsi` alinhadas barra a barra.

## Passo 3 — Backtest com filtro

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_adx_rsi",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_adx_filter"
  }
}
```

## Resultado real

| Variante | Trades | PnL Total | PnL Bruto | Fees | Win% | PF |
|---|---|---|---|---|---|---|
| RSI puro (sem fee) | 135 | +$1.496 | +$1.496 | $0 | 73,3% | 1,228 |
| RSI puro (fee 0,1%) | 135 | -$15.834 | +$1.496 | $17.330 | 9,6% | 0,099 |
| RSI+ADX>25 (sem fee) | 72 | -$147 | -$147 | $0 | 63,9% | 0,963 |
| RSI+ADX>25 (fee 0,1%) | 72 | -$9.381 | -$147 | $9.234 | 9,7% | 0,066 |
| RSI+ADX>30 (fee 0,1%) | 53 | -$5.902 | +$880 | $6.783 | 17,0% | 0,120 |

### Interpretação

- **O filtro reduz trades** (135→72) — bom para taxas, que caem de $17.330 para $9.234.
- **Mas também reduz o edge bruto** (de +$1.496 para –$147) — o filtro eliminou mais sinais bons do que whipsaws.
- **ADX>30 é a melhor variante** com fee: menos trades (53), edge bruto positivo (+$880), mas ainda insuficiente (razão edge/fee = 0,13 ≪ 1,0).
- **Filtro não cria edge** — só remove sinais. Se alguns eram bons, você perdeu edge junto com o ruído.

## Variações

- **ADX como gate direcional:** só compra se DI+ > DI− E ADX > 25
- **RSI mais apertado:** oversold 25, overbought 75
- **Threshold 30:** `ind["adx"][0] > 30.0` (mais restritivo, menos trades)
- **Adicionar ATR para sizing:** `atr(high, low, close, 14)` e ajustar lote por volatilidade
