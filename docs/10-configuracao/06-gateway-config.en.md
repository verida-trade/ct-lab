# WebSocket Gateway Configuration

The market WS gateway (`ct://gateway`) opens automatically when `ct-mcp-server` starts. No manual configuration needed.

To use the ephemeral port in your code:

```json
// Resource: ct://gateway
// Returns: { "ws_url": "ws://127.0.0.1:<port>/<token>", "token": "..." }
```

Connect, send `{ "token": "...", "subscribe": [...] }` and receive live events.

> See [WebSocket Gateway](../07-microestrutura/07-gateway.en.md) for details.
