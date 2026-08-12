# AI Provider Errors

> Common issues with AI providers (OpenAI, Anthropic, Google, Ollama).

## "Model not responding" / Timeout

**Cause:** The provider took longer than the configured timeout.

**Solution:**
1. Check your internet connection
2. Try a faster model (e.g.: `gpt-4o-mini` instead of `gpt-4o`)
3. Restart CT Lab Desktop — this resets the MCP session

## 401 Unauthorized / Invalid API key

```
Error: 401 Unauthorized
```

**Cause:** Incorrect, expired, or unconfigured API key.

**Solution:**
1. Check the key in **Settings → API Keys**
2. For OpenAI: verify at `platform.openai.com/api-keys`
3. For Anthropic: verify at `console.anthropic.com`
4. Re-enter the key (copy without leading/trailing spaces)

## Ollama won't connect

```
Error: connection refused (localhost:11434)
```

**Cause:** Ollama is not running or is on a different port.

**Solution:**
```bash
# Start Ollama
ollama serve

# Check the default port
curl http://localhost:11434/api/tags

# If the port is different, configure via env:
export CT_OLLAMA_URL=http://localhost:11435
```

## Ollama: "function calling not supported"

**Cause:** The chosen model doesn't support function calling (tool use). Without this, the AI cannot invoke MCP tools.

**Solution:** Use models that support tool calling:
- ✅ `llama3.1` (8B+) — supports tool calling
- ✅ `mistral` — supports via specific format
- ❌ `phi3`, `tinyllama` — no tool calling

> For serious ct-mcp-server usage, prefer OpenAI or Anthropic. Ollama is for full privacy, but with limitations.

## Rate limit (429 Too Many Requests)

```
Error: 429 Too Many Requests
```

**Cause:** Provider request limit exceeded.

**Solution:**
1. Reduce request frequency
2. Upgrade your provider plan (e.g.: OpenAI Tier 2+)
3. Use prompts (e.g.: `/backtest`) — guided workflows produce less back-and-forth than free chat

## "Chat mode doesn't execute tools"

**Cause:** `chat` mode disables the agent — the AI only converses, doesn't invoke tools.

**Solution:** Switch to `auto` mode in **Settings → Mode**, or via env:
```bash
export CT_MODE=auto
```

## Google Gemini: "model not found"

**Cause:** Incorrect model name or API key without access to the model.

**Solution:**
1. Check available models at `ai.google.dev`
2. Use `gemini-2.0-flash` (fast) or `gemini-1.5-pro` (powerful)
3. Confirm your key has access to the chosen model

> Back to: [README](./README.en.md)
