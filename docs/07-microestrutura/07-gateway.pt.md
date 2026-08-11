# Gateway WebSocket de Mercado

> `ct://gateway` — visualização realtime (kline, book, trades) via WebSocket.

O servidor abre um `TcpListener` em `127.0.0.1:0` (porta efêmera) e anuncia pelo resource público `ct://gateway`:

```json
// Resource: ct://gateway
{ "ws_url": "ws://127.0.0.1:54321/abc", "token": "..." }
```

## Como funciona

1. O cliente lê `ct://gateway` para obter `ws_url` + `token`
2. Conecta ao WebSocket
3. Envia `{ "token": "...", "subscribe": ["kline:binance:BTCUSDT:1m", "book:binance:BTCUSDT", "trades:binance:BTCUSDT"] }`
4. Recebe snapshot inicial + eventos vivos

## Streams disponíveis

| Stream | Formato |
|---|---|
| `kline:<provider>:<symbol>:<interval>` | Candle vivo |
| `book:<provider>:<symbol>` | Top-N bid/ask + mid + spread |
| `trades:<provider>:<symbol>` | Trades individuais |

> **Fronteira:** série 1s persistida + controle/status seguem MCP. Só o dado vivo de visualização vai pro WS.

---

> Próximo: [Tarefas de coleta](./08-tarefas.pt.md)
