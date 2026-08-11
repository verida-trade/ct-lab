# Collector Status

> Monitor active collections and their states.

## `coletas_ativas`

```json
{ "name": "coletas_ativas", "arguments": {} }
```

Returns:
```json
{ "count": 2, "collectors": [{ "collector_id": "binance:BTCUSDT:trades_1s", "state": "Running" }] }
```

## `parar_coleta`

```json
{ "name": "parar_coleta", "arguments": { "collector_id": "binance:BTCUSDT:trades_1s" } }
```

## Status resources (subscribable)

| URI | Returns |
|---|---|
| `ct://streams/binance/BTCUSDT/trades/status` | Trades worker state |
| `ct://streams/binance/BTCUSDT/book/status` | Book worker state |
| `ct://collectors/status` | Aggregated (subscribable — notifies on transition) |

### `CollectorStatus` states

| State | Description |
|---|---|
| `Starting` | Starting up |
| `Backfilling` | Downloading bulk history |
| `Running` | WS connected and collecting |
| `Reconnecting` | Attempting to reconnect |
| `Stopped` | Stopped |
| `Failed` | Failed |

---

> Next: [Volume & storage](./06-storage.en.md)
