# AI Providers

> Configure which LLM CT Lab uses.

| Provider | Env | Popular models | API Key |
|---|---|---|---|
| OpenAI | `CT_PROVIDER=openai` | `gpt-4o`, `gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `CT_PROVIDER=anthropic` | `claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |
| Google | `CT_PROVIDER=google` | `gemini-2.0-flash`, `gemini-2.0-pro` | `GOOGLE_API_KEY` |
| Ollama (local) | `CT_PROVIDER=ollama` | `llama3`, `mistral` | None (local) |

## Configuration

Via Settings in the desktop app OR via environment variables:

```bash
export CT_PROVIDER=openai
export CT_MODEL=gpt-4o
export OPENAI_API_KEY=sk-...
```

## Ollama (local, no API key)

```bash
# Install Ollama: https://ollama.com
ollama pull llama3
export CT_PROVIDER=ollama
export CT_MODEL=llama3
```

> **Note:** local models may have lower quality for MCP tool usage. Recommended: gpt-4o or claude-sonnet.

---

> Next: [MCP Extensions](./03-extensoes-mcp.en.md)
