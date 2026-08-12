# Collection Errors

> Common issues with microstructure collection (book, trades, klines).

## WebSocket disconnects frequently

**Cause:** Network instability, firewall blocking WebSocket, or Binance rate limit.

**Solution:**
1. Check if your connection is stable
2. Reduce the number of simultaneous symbols (recommended: max 20-50 symbols)
3. Check for firewall/proxy blocking `wss://stream.binance.com`
4. The collector does **automatic reconnect** — if it persists, restart the task:
```python
parar_coleta(id="my_collection")
criar_tarefa_coleta(symbols=["BTCUSDT"], tipos=["trades"])
```

## Collector in "Failed" state

**Cause:** Unrecoverable error — usually invalid WS URL, unavailable provider, or write permission in the streams directory.

**Solution:**
1. Check the detailed error in the status:
```python
listar_tarefas()  # shows state and last error message
```
2. If it's a permission issue: check the Parquet directory:
```bash
# Env var to override the streams directory
export CT_MCP_STREAMS_DIR=/path/to/streams
```
3. Stop and recreate the task:
```python
parar_coleta(id="failed_task")
criar_tarefa_coleta(symbols=["BTCUSDT"], tipos=["book"])
```

## "No data in the last N seconds"

**Cause:** The collector is connected but not receiving messages. The symbol may be inactive or market data is disabled.

**Solution:**
1. Check if the symbol is active on Binance
2. Some symbols have low liquidity — normal for small pairs
3. For book: make sure the symbol has an active order book

## Inconsistent timestamps (ms vs μs)

**Cause:** Binance uses **microsecond** timestamps starting from 2025-01-01 and **millisecond** timestamps before that. The ct-mcp-server normalizes automatically.

**Solution:** Normally transparent. If you use the data via exported Parquet:
- Before 2025-01-01: timestamps in ms (13 digits)
- From 2025-01-01: timestamps in μs (16 digits)

## Trade backfill: "bulk dump limit exceeded"

**Cause:** Binance limits bulk dumps (mass downloads) to a maximum period. The default is 7 days (`backfill_dias: 7`).

**Solution:**
1. Reduce `backfill_dias`:
```python
criar_tarefa_coleta(
    symbols=["BTCUSDT"],
    tipos=["trades"],
    backfill_dias=3  # reduced from 7 to 3
)
```
2. For longer periods, use `buscar_serie_historico` (downloads in successive chunks)

## Klines don't collect in real time

**Cause:** Klines are collected via REST (polling) instead of WebSocket. If the polling interval is too long, there may be gaps.

**Solution:**
1. Klines are collected via `coletar_klines` (REST polling)
2. The default interval is suitable for most cases
3. For real-time candles, consider collecting trades and building klines via pipeline

## Symbol limit for simultaneous collection

**Cause:** Each symbol opens 1-3 WebSocket connections. Too many symbols can overwhelm CPU/RAM.

**Recommendation:**
- ≤ 20 symbols: smooth operation
- 20-50 symbols: monitor CPU
- 50-100 symbols: only on powerful machines (16GB+ RAM)
- > 100 symbols: not recommended

> Back to: [README](./README.en.md)
