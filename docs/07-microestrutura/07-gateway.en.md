# Market WebSocket Gateway

> `ct://gateway` — realtime visualization (kline, book, trades) via WebSocket.

The server opens a `TcpListener` on `127.0.0.1:0` (ephemeral port) and announces via the public resource `ct://gateway`:

```json
// Resource: ct://gateway
{ "ws_url": "ws://127.0.0.1:54321/abc", "token": "..." }
```

## How it works

1. Client reads `ct://gateway` to get `ws_url` + `token`
2. Connects to WebSocket
3. Sends `{ "token": "...", "subscribe": ["kline:binance:BTCUSDT:1m", "book:binance:BTCUSDT", "trades:binance:BTCUSDT"] }`
4. Receives initial snapshot + live events

## Available streams

| Stream | Format |
|---|---|
| `kline:<provider>:<symbol>:<interval>` | Live candle |
| `book:<provider>:<symbol>` | Top-N bid/ask + mid + spread |
| `trades:<provider>:<symbol>` | Individual trades |

> **Boundary:** persisted 1s series + control/status follow MCP. Only live visualization data goes via WS.

---

> Next: [Collection tasks](./08-tarefas.en.md)
