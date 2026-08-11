# Collection Tasks

> Orchestrator that groups collectors under a single scope with a window.

## `criar_tarefa_coleta`

```json
{
  "name": "criar_tarefa_coleta",
  "arguments": {
    "provider": "binance",
    "symbol": "BTCUSDT",
    "escopo": { "trades": true, "book": true, "timeframes": ["15m"] },
    "janela": { "duracao_seg": 3600, "inicio_em": null, "recorrencia_seg": 86400 }
  }
}
```

| Parameter | Description |
|---|---|
| `escopo` | What to collect: trades, book, OHLC timeframes |
| `janela.duracao_seg` | Duration of each session (seconds) |
| `janela.inicio_em` | Start (null = now) |
| `janela.recorrencia_seg` | Repeat every N seconds (optional) |

> **Note:** Tasks live in server memory — they don't persist. The server is a stdio subprocess of the app and terminates together. Scheduling/recurrence only fires with the app open.

## `listar_tarefas` and `parar_tarefa`

```json
{ "name": "listar_tarefas", "arguments": {} }
{ "name": "parar_tarefa", "arguments": { "id": "..." } }
```

---

> Next: [Providers](./09-providers.en.md)
