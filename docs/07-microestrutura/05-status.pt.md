# Status de Coletores

> Monitore coletas ativas e seus estados.

## `coletas_ativas`

```json
{ "name": "coletas_ativas", "arguments": {} }
```

Retorna:
```json
{ "count": 2, "collectors": [{ "collector_id": "binance:BTCUSDT:trades_1s", "state": "Running" }, ...] }
```

## `parar_coleta`

```json
{ "name": "parar_coleta", "arguments": { "collector_id": "binance:BTCUSDT:trades_1s" } }
```

## Resources de status (subscribable)

| URI | O que retorna |
|---|---|
| `ct://streams/binance/BTCUSDT/trades/status` | Estado do trades worker |
| `ct://streams/binance/BTCUSDT/book/status` | Estado do book worker |
| `ct://collectors/status` | Agregado (subscribable — notifica em transição) |

### Estados do `CollectorStatus`

| Estado | Descrição |
|---|---|
| `Starting` | Iniciando |
| `Backfilling` | Baixando bulk histórico |
| `Running` | WS conectado e coletando |
| `Reconnecting` | Tentando reconectar |
| `Stopped` | Parado |
| `Failed` | Falhou |

---

> Próximo: [Volume e storage](./06-storage.pt.md)
