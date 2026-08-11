# Settings do CT Lab

> Onde configurar tudo: provider de IA, modelo, modo, extensões MCP.

## Acessando Settings

No CT Lab Desktop: **Settings** (ícone de engrenagem ou Cmd/Ctrl+,).

## O que configurar

| Seção | O que faz |
|---|---|
| **Provider** | Escolha entre OpenAI, Anthropic, Google, Ollama |
| **Model** | Modelo do provider (ex.: `gpt-4o`, `claude-sonnet-4-20250514`) |
| **Mode** | `auto` (agente decide) ou `chat` (conversa direta) |
| **Extensions** | Adicionar/remover servidores MCP |
| **API Keys** | Chaves dos providers |

## Modo Auto vs Chat

| Modo | Comportamento |
|---|---|
| `auto` | Agente autônomo: lê doutrina, decide, executa tools, explica |
| `chat` | Conversa direta: o modelo só responde, não executa tools |

> Para uso do ct-mcp-server, use `auto` — é o modo que permite ao agente chamar tools.

---

> Próximo: [Providers de IA](./02-providers.pt.md)
