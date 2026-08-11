# Indicadores de Microestrutura

> **Premium.** Indicadores que operam sobre `trades_1s` ou `book_1s`.

| Tool | Input | Janela | O que mede |
|---|---|---|---|
| `tfi` | trades_1s | — | Trade Flow Imbalance (`qty_delta / qty`) |
| `ct_tfi` | trades_1s | `period` | VWMA do TFI ponderado por qty |
| `bfi` | book_1s | — | Book Flow Imbalance |
| `ct_bfi` | book_1s | `period` | VWMA do BFI |
| `obi` | book_1s | — | Order Book Imbalance (top-of-book) |
| `ct_obi` | book_1s | `period` | VWMA do OBI |
| `dbi_01` | book_1s | — | Depth Book Imbalance bin ±0.1% |
| `dbi_1` | book_1s | — | Depth Book Imbalance bin ±1% |
| `mpo` | book_1s | — | Microprice Offset normalizado pelo half-spread |

Todos retornam escala `[-1, +1]`. Janela vazia / sem peso → `0`.

## Exemplo

```json
{
  "name": "ct_tfi",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/trades_1s",
    "period": 60
  }
}
```

## Uso em pipeline

Indicadores de microestrutura podem ser usados na pipeline (`montar_pipeline_indicadores`) e em backtests da mesma forma que indicadores de preço.

---

> Próximo: [Status de coletores](./05-status.pt.md)
