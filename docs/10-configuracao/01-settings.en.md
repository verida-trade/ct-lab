# CT Lab Settings

> Where to configure everything: AI provider, model, mode, MCP extensions.

## Accessing Settings

In CT Lab Desktop: **Settings** (gear icon or Cmd/Ctrl+,).

## What to configure

| Section | What it does |
|---|---|
| **Provider** | Choose between OpenAI, Anthropic, Google, Ollama |
| **Model** | Provider model (e.g., `gpt-4o`, `claude-sonnet-4-20250514`) |
| **Mode** | `auto` (agent decides) or `chat` (direct conversation) |
| **Extensions** | Add/remove MCP servers |
| **API Keys** | Provider keys |

## Auto vs Chat mode

| Mode | Behavior |
|---|---|
| `auto` | Autonomous agent: reads doctrine, decides, executes tools, explains |
| `chat` | Direct conversation: model only responds, doesn't execute tools |

> For ct-mcp-server usage, use `auto` — it's the mode that allows the agent to call tools.

---

> Next: [AI Providers](./02-providers.en.md)
