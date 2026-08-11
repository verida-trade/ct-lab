# 03 — MCP Connection

Now that you have **CT Lab Desktop** installed (step 01) and an **AI provider**
configured (step 02), it's time to understand the MCP connection. The
**ct-mcp-server** ships bundled with CT Lab Desktop — no separate installation
required. This document shows how the server connects to CT Lab so the AI
can use all the tools.

---

## Table of Contents

- [How the connection works](#how-the-connection-works)
- [Connecting via UI (Settings)](#connecting-via-ui-settings)
- [What happens behind the scenes](#what-happens-behind-the-scenes)
- [Verifying the connection](#verifying-the-connection)
- [Common issues](#common-issues)
- [Server status](#server-status)

---

## How the connection works

CT Lab Desktop acts as the **MCP host** — it manages the lifecycle of the
`ct-mcp-server` bundled with the app. When you send a message in the chat,
the flow is:

```
  You type: "Fetch BTCUSDT 15m and calculate RSI"
        │
        ▼
  ┌─ CT Lab Desktop ───────────────────────────────────┐
  │                                                   │
  │   1. Sends your request to the AI provider        │
  │      (OpenAI / Anthropic / Google / Ollama)      │
  │                                                   │
  │   2. The AI decides to call MCP tools:            │
  │      • buscar_serie("BTCUSDT", "15m")             │
  │      • rsi(uri, period=14)                        │
  │                                                   │
  │   3. CT Lab forwards the calls via stdin to       │
  │      the ct-mcp-server subprocess                  │
  │                                                   │
  │   4. ct-mcp-server executes and returns result     │
  │      via stdout                                   │
  │                                                   │
  │   5. AI receives the result and formulates reply  │
  │                                                   │
  └───────────────────────────────────────────────────┘
```

The AI never talks directly to `ct-mcp-server`. CT Lab Desktop is the
intermediary: it receives tool requests from the AI, forwards them to the
subprocess, and returns the responses.

---

## Connecting via UI (Settings)

To link ct-mcp-server to CT Lab:

### Step 1 — Open extensions

1. Open CT Lab Desktop.
2. Go to **Settings** (`Cmd/Ctrl + ,`).
3. Click the **Extensions** tab.

### Step 2 — Add MCP Server

1. Click **Add MCP Server**.
2. Fill in the form:

```
┌───────────────────────────────────────────────────────────────────────┐
│  Add MCP Server                                                       │
│                                                                       │
│  Name:             [ct-mcp-server          ]                          │
│  Type:             [stdio                  ▼]                        │
│  Binary path:     [~/ct-mcp/ct-mcp-server  ]                          │
│  Args:             [                        ]                          │
│  Env vars:         [CT_PROVIDER=openai      ]                          │
│                   [CT_MODEL=gpt-4o          ]                          │
│                   [OPENAI_API_KEY=sk-•••••  ]                          │
│                   [CT_MODE=auto             ]                          │
│                                                                       │
│           [Test]    [Cancel]    [Save]                                 │
└───────────────────────────────────────────────────────────────────────┘
```

| Field | Value |
|-------|-------|
| **Name** | `ct-mcp-server` (or any name you prefer) |
| **Type** | `stdio` (always — the server uses stdio, not network) |
| **Binary path** | Full path to the binary (e.g., `~/ct-mcp/ct-mcp-server`) |
| **Args** | Leave empty in most cases |
| **Env vars** | AI provider environment variables (see doc 02) |

3. Click **Test**. CT Lab will launch the subprocess and verify basic
   resources respond.

> ✅ If the test passes, you'll see: `Connection successful — 122 tools available`.

4. Click **Save**.

### Step 3 — Confirm status

In the extensions list, you should see:

```
  ┌────────────────────────────────────────────────────────────────────┐
  │ Extensions                                                          │
  │                                                                    │
  │  ●  ct-mcp-server              stdio    Connected    122 tools      │
  │                                                                    │
  └────────────────────────────────────────────────────────────────────┘
```

The green **Connected** indicator confirms the subprocess is active and
responding.

---

## What happens behind the scenes

When you save the configuration, CT Lab Desktop:

1. **Starts the subprocess** — the binary `~/ct-mcp/ct-mcp-server` is launched
   with the configured environment variables.

2. **Establishes the stdio channel** — communication uses stdin/stdout
   (JSON-RPC over stdio). No network ports.

3. **Discovers tools** — CT Lab queries which tools the server exposes (e.g.,
   `buscar_serie`, `rsi`, `sma`, `ct_backtest`, etc.).

4. **Registers tools with the AI** — CT Lab sends the tool list to the AI
   provider, so it knows what it can call.

5. **Keeps the subprocess alive** — if the subprocess terminates unexpectedly,
   CT Lab restarts it automatically.

```
  CT Lab Desktop
       │
       ├── spawn → ~/ct-mcp/ct-mcp-server (stdio subprocess)
       │              │
       │              ├── stdin  ← JSON-RPC (tool calls)
       │              ├── stdout → JSON-RPC (responses)
       │              └── stderr → logs
       │
       └── HTTP/WS → AI Provider (OpenAI / Anthropic / Google / Ollama)
```

---

## Verifying the connection

### Quick chat test

Open the chat in CT Lab and type:

> **"List the available time series."**

What happens:

1. The AI receives your request.
2. The AI decides to call the `listar_series` (JSON-RPC) / `listarSeries` (TypeScript SDK) tool.
3. CT Lab forwards the call to ct-mcp-server.
4. The server returns cached series (or an empty list if it's the first time).
5. The AI presents the result in natural language.

**Expected response:**

```text
🤖 No series found in cache. To fetch a series, just ask me —
   for example: "Fetch BTCUSDT in 15m from Binance."
```

### Advanced test

To confirm market data is accessible, type:

> **"Fetch BTCUSDT in 15m from Binance and tell me how many candles were loaded."**

The AI will call `buscar_serie` (JSON-RPC) / `buscarSerie` (TypeScript SDK) with
the parameters, and the server will fetch data from Binance:

```text
🤖 I fetched the BTCUSDT series at 15m from Binance.
   • URI: ct://series/binance/BTCUSDT/15m
   • Candles loaded: 500
   • First candle: 2026-08-10 00:00 UTC
   • Last candle: 2026-08-11 15:45 UTC
```

---

## Common issues

| Problem | Likely cause | Solution |
|---------|-------------|----------|
| Status **Disconnected** | Incorrect binary path | Check the **Binary path** field in Extensions |
| `0 tools available` | Server started but didn't register tools | Restart the server: click **Restart** in Extensions |
| AI doesn't call tools | `CT_MODE` is set to `chat` | Change to `auto` in **Settings → AI Provider** |
| `Tool not found` | Outdated server | Update CT Lab Desktop to the latest version |
| `Connection timeout` | AI provider not responding | Check API key and internet connection |
| Resource `ct://host/fingerprint` not responding | Server not connected | Redo the configuration in Extensions |

---

## Server status

At any time, you can check the server status:

| Method | How to check |
|--------|-------------|
| UI | **Settings → Extensions** — green/red indicator |
| MCP resource | `ct://host/fingerprint` — shows the digital signature |
| Logs | **Settings → Advanced → Open Logs** — subprocess stderr |

> 💡 If something stops working, **Restart** in Extensions restarts the
> subprocess without closing CT Lab.

---

## Next Steps

- ➡️ **[04 — First Project](./04-primeiro-projeto)** — Run your first end-to-end backtest
- ⬅️ **[02 — AI Provider](./02-provider-ia)** — Back to AI configuration
- ⬅️ **[Index](./README)**

---

_Last updated: 2026-08-11_
