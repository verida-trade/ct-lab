# CT Lab Ecosystem Overview

> **Folder:** `docs/02-conceitos/01-visao-geral.en.md`  
> **Related reading:** [`02-quatro-camadas`](./02-quatro-camadas.en.md) ·
> [`05-mcp-protocolo`](./05-mcp-protocolo.en.md)  
> **Target audience:** beginners and integrators

---

## What is CT Lab?

CT Lab is a quantitative analysis and algorithmic trading platform that runs
locally on the user's machine. It exposes a set of tools for discovering,
ingesting, transforming, and backtesting financial series — all accessible to
an LLM (Large Language Model) via **MCP** (Model Context Protocol).

Core premise: the user converses naturally with the AI, and the AI uses MCP
tools to read data, compute indicators, run backtests, and build pipelines —
without the user needing to write code.

---

## Component Diagram

```
 ┌───────────────────────────────────────────────────────────────┐
 │                        User                                    │
 │             (natural language conversation)                    │
 └───────────────────────┬───────────────────────────────────────┘
                         │  "What's the 20-period SMA of BTCUSDT 1h?"
                         ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                  CT Lab Desktop (Electron UI)                  │
 │  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐  │
 │  │   Chat UI    │    │  Charts /    │    │  Account         │  │
 │  │   (Markdown)  │    │  Tables      │    │  Free / Premium  │  │
 │  └──────┬───────┘    └──────────────┘    └──────────────────┘  │
 └─────────┼─────────────────────────────────────────────────────┘
           │  MCP stdio (JSON-RPC)
           ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                    ct-mcp-server                               │
 │                                                               │
 │   Tools (actions: buscar_serie, sma, ct_backtest, …)           │
 │   Resources (reads via templates: tail/head/sample)            │
 │   Prompts (guided workflows: saudacao, backtest, …)            │
 │   Completions (autocomplete prompt arguments)                  │
 └─────────┬─────────────────────────────────────────────────────┘
           │  HTTP (localhost)
           ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                    ct-labd (local HTTP server)                 │
 │                                                               │
 │   ┌──────────┐   ┌───────────┐   ┌───────────┐   ┌────────┐  │
 │   │ Series   │   │Indicators │   │ Backtest  │   │  ML    │  │
 │   │ Repo     │   │ Engine    │   │ Engine    │   │Engine  │  │
 │   └──────────┘   └───────────┘   └───────────┘   └────────┘  │
 │                                                               │
 │   Cache (up to 1 series Free · up to 100 Premium)              │
 │   Providers: Binance, Yahoo Finance, CSV, …                    │
 └───────────────────────────────────────────────────────────────┘
           │
           ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                       LLM Provider                             │
 │     (OpenAI / Anthropic / Google / Local)                      │
 │     Receives tool definitions, invokes tools, reasons          │
 └───────────────────────────────────────────────────────────────┘
```

---

## Data Flow — Step by Step

1. **The user asks a question** in natural language in the Chat UI.
2. **The LLM receives** the question, conversation history, and MCP tool
   definitions exposed by `ct-mcp-server`.
3. **The LLM decides** whether to call a tool — e.g., `buscar_serie` to
   locate the series `ct://series/binance/BTCUSDT/1h`.
4. **The `ct-mcp-server`** forwards the call via HTTP to `ct-labd`, which
   checks the cache or queries the provider (Binance, Yahoo, etc.).
5. **The tool returns** a URI + metadata — never raw data rows directly.
6. **The LLM reads the data** via a resource template, e.g.
   `ct://series/binance/BTCUSDT/1h/tail/20` for the last 20 rows.
7. **The LLM computes** additional indicators if needed (e.g., `sma`), and
   synthesizes a natural language response.
8. **The user sees** the rendered response in the Chat UI, with optional
   charts and tables.

---

## Components Summary

| Component | Technology | Role |
|-----------|------------|------|
| **CT Lab Desktop** | Electron + Web UI | User graphical interface (chat, charts, account management) |
| **ct-mcp-server** | MCP stdio (JSON-RPC) | MCP bridge between LLM and local backend |
| **ct-labd** | HTTP server (localhost) | Data, indicator, backtest, and ML engine |
| **LLM Provider** | OpenAI / Anthropic / etc. | Reasoning and tool invocation |
| **Data providers** | Binance, Yahoo, CSV | External OHLCV series sources |

---

## Why MCP?

MCP (Model Context Protocol) is an open standard that lets LLMs discover and
invoke external tools in a structured way. In CT Lab, MCP solves three
problems:

| Problem | MCP Solution |
|---------|-------------|
| How does the AI find data? | **Resources** — `ct://` URIs with `tail/head` templates |
| How does the AI execute actions? | **Tools** — MCP functions returning URI + metadata |
| How does the user start workflows? | **Prompts** — guided templates invoked by the user |

See [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) for the full deep dive.

---

## Installation and First Run

```bash
# Clone the public repository
git clone https://github.com/verida-trade/ct-lab.git
cd ct-lab

# Install dependencies (Node.js ≥ 20)
npm install

# Start CT Lab Desktop
npm start
```

After starting, CT Lab Desktop:
1. Launches `ct-labd` on a local port (e.g., `http://localhost:8420`).
2. Launches `ct-mcp-server` in stdio mode.
3. Connects to the configured LLM (API key via environment variable).

---

## Quick Verification

After first launch, test with a simple question in the chat:

```
  User: What's the current price of BTCUSDT on Binance?

  AI: → calls buscar_serie(provider="binance", symbol="BTCUSDT", timeframe="1m")
     → receives ct://series/binance/BTCUSDT/1m
     → reads ct://series/binance/BTCUSDT/1m/tail/1
     → responds with the current price
```

If you see the returned URI and the price in the response, the ecosystem is
working correctly.

---

## Next Steps

- [`02-quatro-camadas`](./02-quatro-camadas.en.md) — the 4-layer architecture
- [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) — how MCP works in CT
- [`06-free-vs-premium`](./06-free-vs-premium.en.md) — plans and licensing
