# Consulta de Dados L2

> `consultar_trades` e `consultar_book` — query agregada de microestrutura.

Estas tools retornam **URI + meta + estatísticas bounded** — o modelo lê dados ponto-a-ponto via resources, não via tool.

## `consultar_trades`

```json
{
  "name": "consultar_trades",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/trades_1s",
    "agregacao": "1h"
  }
}
```

Retorna: `{ uri, agregacao, row_count, first_ts, last_ts, buckets, summary }`

## `consultar_book`

```json
{
  "name": "consultar_book",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/book_1s",
    "agregacao": "5m"
  }
}
```

## Leitura ponto-a-ponto

O modelo lê os dados via resource templates:

```
ct://series/binance/BTCUSDT/trades_1s/tail/100
ct://series/binance/BTCUSDT/book_1s/tail/100
```

---

> Próximo: [Indicadores de microestrutura](./04-indicadores-micro.pt.md)
