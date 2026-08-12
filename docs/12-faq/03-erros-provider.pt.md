# Erros de Provider de IA

> Problemas comuns com provedores de IA (OpenAI, Anthropic, Google, Ollama).

## "Model not responding" / Timeout

**Causa:** O provedor demorou mais que o timeout configurado.

**Solução:**
1. Verifique a conexão com internet
2. Tente um modelo mais rápido (ex.: `gpt-4o-mini` em vez de `gpt-4o`)
3. Reinicie o CT Lab Desktop — isso reinicia a sessão MCP

## 401 Unauthorized / Invalid API key

```
Error: 401 Unauthorized
```

**Causa:** Chave de API incorreta, expirada ou não configurada.

**Solução:**
1. Verifique a chave em **Settings → API Keys**
2. Para OpenAI: confirme em `platform.openai.com/api-keys`
3. Para Anthropic: confirme em `console.anthropic.com`
4. Reinsira a chave (copie sem espaços no início/fim)

## Ollama não conecta

```
Error: connection refused (localhost:11434)
```

**Causa:** Ollama não está rodando ou está em outra porta.

**Solução:**
```bash
# Inicie o Ollama
ollama serve

# Verifique a porta padrão
curl http://localhost:11434/api/tags

# Se a porta for diferente, configure via env:
export CT_OLLAMA_URL=http://localhost:11435
```

## Ollama: "function calling not supported"

**Causa:** O modelo escolhido não suporta function calling (tool use). Sem isso, a IA não consegue invocar ferramentas MCP.

**Solução:** Use modelos que suportam tool calling:
- ✅ `llama3.1` (8B+) — suporta tool calling
- ✅ `mistral` — suporta via formato específico
- ❌ `phi3`, `tinyllama` — sem tool calling

> Para uso sério do ct-mcp-server, prefira OpenAI ou Anthropic. Ollama é para privacidade total, mas com limitações.

## Rate limit (429 Too Many Requests)

```
Error: 429 Too Many Requests
```

**Causa:** Limite de requisições do provedor excedido.

**Solução:**
1. Reduza a frequência de pedidos
2. Faça upgrade do plano do provedor (ex.: OpenAI Tier 2+)
3. Use prompts (ex.: `/backtest`) — guiados produzem menos ida-e-volta que chat livre

## "Modo chat não executa tools"

**Causa:** O modo `chat` desliga o agente — a IA só conversa, não invoca ferramentas.

**Solução:** Troque para modo `auto` em **Settings → Mode**, ou via env:
```bash
export CT_MODE=auto
```

## Google Gemini: "model not found"

**Causa:** Nome do modelo incorreto ou API key sem acesso ao modelo.

**Solução:**
1. Verifique modelos disponíveis em `ai.google.dev`
2. Use `gemini-2.0-flash` (rápido) ou `gemini-1.5-pro` (potente)
3. Confirme que a chave tem acesso ao modelo escolhido

> Voltar para: [README](./README.pt.md)
