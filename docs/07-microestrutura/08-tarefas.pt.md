# Tarefas de Coleta

> Orquestrador que agrupa coletores num escopo único sob uma janela.

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

| Parâmetro | Descrição |
|---|---|
| `escopo` | O que coletar: trades, book, timeframes OHLC |
| `janela.duracao_seg` | Duração de cada sessão (segundos) |
| `janela.inicio_em` | Início (null = agora) |
| `janela.recorrencia_seg` | Repetir a cada N segundos (opcional) |

> **Nota:** Tarefas vivem em memória no servidor — não persistem. O servidor é subprocesso stdio do app e encerra junto. Agendamento/recorrência só disparam com o app aberto.

## `listar_tarefas` e `parar_tarefa`

```json
{ "name": "listar_tarefas", "arguments": {} }
{ "name": "parar_tarefa", "arguments": { "id": "..." } }
```

---

> Próximo: [Providers](./09-providers.pt.md)
