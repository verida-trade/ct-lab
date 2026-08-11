# Coleta de Trades (1 segundo)

> **Premium.** `coletar_trades` — colete dados de trades em tempo real com 1 segundo de granularidade.

## Chamada

```json
{
  "name": "coletar_trades",
  "arguments": {
    "provider": "binance",
    "symbol": "BTCUSDT",
    "backfill_dias": 7
  }
}
```

| Parâmetro | Descrição |
|---|---|
| `provider` | `"binance"` (Spot) ou `"binance_um"` (Futures USDⓈ-M) |
| `symbol` | Símbolo (ex.: `"BTCUSDT"`) |
| `backfill_dias` | N dias de backfill bulk (opcional; baixa dumps diários da Binance) |

## Retorno

```json
{
  "collector_id": "binance:BTCUSDT:trades_1s",
  "status": "started",
  "started_at": 1784951100
}
```

## Schema `trades_1s` (41 colunas)

| Grupo | Colunas |
|---|---|
| Contagem (4) | `n_trades`, `n_buys`, `n_sells`, `n_delta` |
| Volume base (4) | `qty`, `buy_qty`, `sell_qty`, `qty_delta` |
| Volume quote (4) | `quote_qty`, `buy_quote_qty`, `sell_quote_qty`, `quote_delta` |
| Preço (5) | `vwap`, `price_open`, `price_high`, `price_low`, `price_close` |
| Cursor (2) | `first_agg_id`, `last_agg_id` |
| Distribuição qty (11) | min, max, mean, std, p01, p10, p25, p50, p75, p90, p99 |
| Distribuição quote (11) | idem |

## Segundos vazios

Segundos sem trades: `n_*=0`, volumes=0, preços=NaN, cursores=NaN. **Linha emitida normalmente** — gaps explícitos.

## URI

```
ct://series/binance/BTCUSDT/trades_1s
```

---

> Próximo: [Coleta de book](./02-coletar-book.pt.md)
