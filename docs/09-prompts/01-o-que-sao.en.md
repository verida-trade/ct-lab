# What Are MCP Prompts

> Prompts are guided workflows that the **user** invokes — not the model.

In the MCP protocol, prompts are distinct from tools and resources:

| Primitive | Who uses it | What it does |
|---|---|---|
| **Tools** | Model | Actions (buscar_serie, sma, ct_backtest) |
| **Resources** | Model | Data reads (ct://series/.../tail/10) |
| **Prompts** | User | Injects messages that seed a workflow |

## Mechanics

1. **`prompts/list`** — returns the list of available prompts
2. **`prompts/get(name, args)`** — injects messages into the chat that seed the task
3. The model then executes the workflow using tools and resources

> **Who triggers is the user/host, not the model.** The model uses tools and resources; the user invokes prompts.

---

> Next: [Public prompts](./02-prompts-publicos.en.md)
