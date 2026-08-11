# Coleta de Order Book (1 segundo)

> **Premium.** `coletar_book` — reconstrua o order book localmente e aggregate em 1 segundo.

## Chamada

```json
{
  "name": "coletar_book",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT" }
}
```

> Book não tem `backfill_dias` — Spot não tem bulk L2; Futures `bookDepth/` é agregado ±%, não reconstrói.

## Schema `book_1s` (51 colunas)

Mesmas 40 colunas do `trades_1s` (adaptadas) + 11 extras book-only.

### Mapeamento (40 colunas simétricas)

| `trades_1s` | `book_1s` |
|---|---|
| `n_trades` | `n_updates` |
| `n_buys`/`n_sells` | `n_placements`/`n_cancellations` |
| `qty` | `qty_change_total` |
| `buy_qty`/`sell_qty` | `qty_placed`/`qty_removed` |
| `price_o/h/l/c` | `mid_o/h/l/c` |

### Extras book-only (11)

`bid_price`, `ask_price`, `bid_qty_top`, `ask_qty_top`, `spread_bps`, `bid_qty_delta`, `ask_qty_delta`, `depth_0_1pct_bid`, `depth_0_1pct_ask`, `depth_1pct_bid`, `depth_1pct_ask`

## Segundos vazios (book)

State cols carregam último valor (não viram NaN — representariam "book sumiu"). `mid_o/h/l/c` = `mid_close` anterior.

## URI

```
ct://series/binance/BTCUSDT/book_1s
```

---

> Próximo: [Consulta de dados](./03-consulta.pt.md)
