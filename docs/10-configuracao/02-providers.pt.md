# Providers de IA

> Configure qual LLM o CT Lab usa.

| Provider | Env | Modelos populares | API Key |
|---|---|---|---|
| OpenAI | `CT_PROVIDER=openai` | `gpt-4o`, `gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `CT_PROVIDER=anthropic` | `claude-sonnet-4-20250514`, `claude-haiku-4-20250422` | `ANTHROPIC_API_KEY` |
| Google | `CT_PROVIDER=google` | `gemini-2.0-flash`, `gemini-2.0-pro` | `GOOGLE_API_KEY` |
| Ollama (local) | `CT_PROVIDER=ollama` | `llama3`, `mistral`, `qwen2` | Nenhuma (local) |

## Configuração

Via Settings no desktop OU via variáveis de ambiente:

```bash
export CT_PROVIDER=openai
export CT_MODEL=gpt-4o
export OPENAI_API_KEY=sk-...
```

## Ollama (local, sem API key)

```bash
# Instalar Ollama: https://ollama.com
ollama pull llama3
export CT_PROVIDER=ollama
export CT_MODEL=llama3
```

> **Nota:** modelos locais podem ter qualidade menor para uso com tools MCP. Recomendado: gpt-4o ou claude-sonnet.

---

> Próximo: [Extensões MCP](./03-extensoes-mcp.pt.md)
