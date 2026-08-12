# Configuração do Gateway WebSocket

O gateway WS de mercado (`ct://gateway`) abre automaticamente quando o `ct-mcp-server` inicia. Não há configuração manual necessária.

Para usar a porta efêmera no seu código:

```json
// Resource: ct://gateway
// Retorna: { "ws_url": "ws://127.0.0.1:<porta>/<token>", "token": "..." }
```

Conecte, envie `{ "token": "...", "subscribe": [...] }` e receba eventos vivos.

> Veja [Gateway WebSocket](../07-microestrutura/07-gateway.pt.md) para detalhes.
