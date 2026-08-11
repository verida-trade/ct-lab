# 02 — AI Provider

CT Lab works with multiple AI providers. The AI is the "brain" that interprets
your natural-language requests and decides which ct-mcp-server tools to call.
This document shows how to configure each supported provider.

---

## Table of Contents

- [Overview](#overview)
- [Environment variables](#environment-variables)
- [OpenAI](#openai)
- [Anthropic](#anthropic)
- [Google AI](#google-ai)
- [Ollama (local)](#ollama-local)
- [Operating modes (CT_MODE)](#operating-modes-ct_mode)
- [Configuration via UI](#configuration-via-ui)
- [Verification](#verification)
- [Switching providers](#switching-providers)
- [Troubleshooting](#troubleshooting)

---

## Overview

CT Lab supports four AI providers:

| Provider | Type | Requires internet | Popular models |
|----------|------|-------------------|----------------|
| OpenAI | Cloud | ✅ | gpt-4o, gpt-4o-mini |
| Anthropic | Cloud | ✅ | claude-sonnet-4-20250514, claude-haiku |
| Google | Cloud | ✅ | gemini-2.0-flash, gemini-1.5-pro |
| Ollama | Local | ❌ | llama3, mistral, phi3, … |

> 💡 **Recommended for beginners**: OpenAI gpt-4o or Anthropic
> claude-sonnet-4-20250514 — both have excellent MCP protocol support.

> 🔒 **Full privacy**: if you don't want to send data to the cloud, use Ollama
> locally.

---

## Environment variables

All provider configuration can be done via environment variables or the CT
Lab UI. The three main variables are:

| Variable | Description | Example |
|----------|-------------|---------|
| `CT_PROVIDER` | Identifies the AI provider | `openai`, `anthropic`, `google`, `ollama` |
| `CT_MODEL` | Specifies the model within the provider | `gpt-4o`, `claude-sonnet-4-20250514`, `llama3` |
| `CT_MODE` | Operating mode | `auto` (default) or `chat` |

### API Keys

Each cloud provider requires an API key:

| Variable | Provider | Where to get it |
|----------|----------|-----------------|
| `OPENAI_API_KEY` | OpenAI | platform.openai.com/api-keys |
| `ANTHROPIC_API_KEY` | Anthropic | console.anthropic.com |
| `GOOGLE_API_KEY` | Google | aistudio.google.com/apikey |

> ⚠️ **Never share your API keys.** CT Lab stores keys locally and never
> sends them to third-party servers.

---

## OpenAI

### Environment variable configuration

```bash
export CT_PROVIDER=openai
export CT_MODEL=gpt-4o
export OPENAI_API_KEY="sk-your-key-here"
export CT_MODE=auto
```

### UI configuration

1. Open CT Lab → **Settings → AI Provider**.
2. Select **OpenAI** from the dropdown.
3. Enter your API key in the **API Key** field.
4. Select the **gpt-4o** model (or another available model).
5. Set the mode to **Auto** (recommended).
6. Click **Save**.

### Verify

```bash
# In your terminal, before opening CT Lab:
echo $CT_PROVIDER   # should print: openai
echo $CT_MODEL       # should print: gpt-4o
```

---

## Anthropic

### Environment variable configuration

```bash
export CT_PROVIDER=anthropic
export CT_MODEL=claude-sonnet-4-20250514
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
export CT_MODE=auto
```

### UI configuration

1. Open CT Lab → **Settings → AI Provider**.
2. Select **Anthropic** from the dropdown.
3. Enter your API key in the **API Key** field.
4. Select the **claude-sonnet-4-20250514** model (or another available model).
5. Set the mode to **Auto** (recommended).
6. Click **Save**.

### Verify

```bash
echo $CT_PROVIDER   # should print: anthropic
echo $CT_MODEL       # should print: claude-sonnet-4-20250514
```

---

## Google AI

### Environment variable configuration

```bash
export CT_PROVIDER=google
export CT_MODEL=gemini-2.0-flash
export GOOGLE_API_KEY="your-key-here"
export CT_MODE=auto
```

### UI configuration

1. Open CT Lab → **Settings → AI Provider**.
2. Select **Google** from the dropdown.
3. Enter your API key in the **API Key** field.
4. Select the **gemini-2.0-flash** model (or another available model).
5. Set the mode to **Auto** (recommended).
6. Click **Save**.

### Verify

```bash
echo $CT_PROVIDER   # should print: google
echo $CT_MODEL       # should print: gemini-2.0-flash
```

---

## Ollama (local)

Ollama lets you run AI models locally without internet. It's ideal for total
privacy or offline environments.

### Prerequisites

1. Install Ollama: [ollama.com](https://ollama.com)
2. Pull a model:

```bash
ollama pull llama3
```

### Environment variable configuration

```bash
export CT_PROVIDER=ollama
export CT_MODEL=llama3
export CT_MODE=auto
```

> ⚠️ No API key is required for local Ollama.

### UI configuration

1. Open CT Lab → **Settings → AI Provider**.
2. Select **Ollama** from the dropdown.
3. In the **Model** field, type `llama3` (or another downloaded model).
4. Set the mode to **Auto** (recommended).
5. Click **Save**.

### Verify

```bash
# Confirm Ollama is running
ollama list

# Should list:
# NAME       SIZE    MODIFIED
# llama3     4.7 GB  2 hours ago
```

> 💡 **Recommended models for MCP use**: `llama3` (8B), `mistral` (7B),
> `phi3` (3.8B — lighter). Larger models (13B+) require more RAM/GPU.

---

## Operating modes (CT_MODE)

| Mode | Description | When to use |
|------|-------------|-------------|
| `auto` | The AI automatically decides when and which tools to call | ✅ Recommended for most cases |
| `chat` | The AI only converses without automatically calling tools | For conceptual discussions without executing actions |

In `auto` mode, the AI:

1. Receives your natural-language request.
2. Decides which MCP tool to call (e.g., `buscar_serie` / `buscarSerie`).
3. Executes the call via ct-mcp-server.
4. Analyzes the result.
5. Calls more tools if needed (e.g., `rsi`, `ct_backtest` / `ctBacktest`).
6. Presents the final result in natural language.

---

## Configuration via UI

All environment variables above can also be set in the UI:

```
┌───────────────────────────────────────────────────────────────────────┐
│  Settings > AI Provider                                               │
│                                                                       │
│  Provider:        [OpenAI    ▼]                                       │
│  Model:           [gpt-4o    ▼]                                       │
│  API Key:         [sk-•••••••••••••••••••••••••••••••••••]           │
│  Mode:            [Auto      ▼]                                       │
│                                                                       │
│           [Test Connection]    [Save]                                  │
└───────────────────────────────────────────────────────────────────────┘
```

Use the **Test Connection** button to verify the AI responds correctly before
saving.

---

## Verification

After configuring, open the chat in CT Lab and type:

> "Hello! List the available time series."

If everything is correct, the AI will respond with a list of series — which
means the connection to both the provider and ct-mcp-server is working.

---

## Switching providers

To switch providers:

1. Go to **Settings → AI Provider**.
2. Select the new provider.
3. Enter the new API key (if applicable).
4. Select the model.
5. Click **Test Connection** and then **Save**.

> No need to restart CT Lab. The change takes effect immediately.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `Authentication failed` | Verify your API key is correct and valid |
| `Model not found` | Confirm the exact model name (e.g., `gpt-4o`, not `gpt4`) |
| Ollama not responding | Check the service is active: `ollama serve` |
| Responses without tools | Change `CT_MODE` from `chat` to `auto` |
| Timeout | Check your internet connection (or local Ollama load) |

---

## Next Steps

- ➡️ **[03 — MCP Connection](./03-conexao-mcp)** — Connect ct-mcp-server to CT Lab
- ⬅️ **[01 — CT Lab Desktop](./01-ct-lab-desktop)** — Back to app installation
- ⬅️ **[Index](./README)**

---

_Last updated: 2026-08-11_
