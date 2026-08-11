# O Que São Prompts MCP

> Prompts são workflows guiados que o **usuário** invoca — não o modelo.

No protocolo MCP, prompts são distintos de tools e resources:

| Primitiva | Quem usa | O que faz |
|---|---|---|
| **Tools** | Modelo | Ações (buscar_serie, sma, ct_backtest) |
| **Resources** | Modelo | Leitura de dados (ct://series/.../tail/10) |
| **Prompts** | Usuário | Injeta mensagens que semeiam um workflow |

## Mecânica

1. **`prompts/list`** — retorna a lista de prompts disponíveis
2. **`prompts/get(nome, args)`** — injeta mensagens no chat que semeiam a tarefa
3. O modelo então executa o workflow usando tools e resources

> **Quem dispara é o usuário/host, não o modelo.** O modelo usa tools e resources; o usuário invoca prompts.

---

> Próximo: [Prompts públicos](./02-prompts-publicos.pt.md)
