# Installation — Complete Guide

Welcome to **CT Lab**! This section walks you step by step from downloading the
desktop application to running your first quantitative analysis project with AI.

---

## Prerequisites

| Item | Details |
|------|---------|
| Operating system | macOS 11 (Big Sur) or later · Ubuntu 20.04+ · Windows 10+ |
| Internet | Required for download, activation, and market-data queries |
| AI provider account | OpenAI, Anthropic, Google AI — or Ollama running locally |
| API key | From your exchange (Binance) or data provider (Yahoo Finance), if applicable |

---

## Installation Roadmap

Follow the documents in the order below:

| # | Document | What you'll do |
|---|----------|----------------|
| 1 | [CT Lab Desktop](./01-ct-lab-desktop) | Download and install the app from verida.trade |
| 2 | [AI Provider](./02-provider-ia) | Configure OpenAI, Anthropic, Google, or Ollama |
| 3 | [MCP Connection](./03-conexao-mcp) | Connect CT Lab Desktop to ct-mcp-server |
| 4 | [First Project](./04-primeiro-projeto) | Fetch data, compute indicators, and run a backtest |

> 💡 **Tip**: If you're experienced and want to jump straight in, skip to
> **04 — First Project**. Otherwise, we recommend following the sequential order.

---

## Architecture Overview

CT Lab works with three main components:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CT LAB DESKTOP (Electron)                                │
│                                                                                  │   │
│   ┌─ AI Chat Interface ─────────┐         ┌─ Settings & Extensions ─────────────┐  │
│   │  • Conversation panel       │         │  • AI Provider (OpenAI, Claude…)    │  │
│   │  • Chart rendering          │         │  • MCP Server (stdio subprocess)   │  │
│   │  • Backtest results         │         │  • License & premium subscription   │  │
│   └─────────────────────────────┘         └─────────────────────────────────────┘  │
│                        │                                            │       │
│                        ▼                                            ▼       │
│   ┌─────────────────────────────┐         ┌──────────────────────────────────────┐ │
│   │    AI Provider (cloud)      │         │        ct-mcp-server (local)          │ │
│   │  OpenAI · Anthropic · Google │         │  • CT Tools (buscar_serie,             │ │
│   │  · Ollama (local)            │         │    rsi, sma, backtest, …)             │ │
│   └─────────────────────────────┘         │  • Time-series cache                  │ │
│                                             │  • Technical & CT indicators          │ │
│                                             └──────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

1. **CT Lab Desktop** — Electron app available for macOS, Linux, and Windows.
   This is where you chat with the AI, visualize charts, and configure everything.
2. **ct-mcp-server** — local process (stdio) bundled with CT Lab Desktop that
   exposes all CT Lab tools (series fetching, technical indicators, backtests,
   etc.) to the AI. No separate installation required.
3. **AI Provider** — the service that generates responses and decides which
   tools to call (OpenAI, Anthropic, Google, or local Ollama).

---

## Licensing

CT Lab offers two plans:

| Feature | Free | Premium |
|---------|------|---------|
| Time-series cache | 1 series | 100 series |
| Public technical indicators | 36 | 36 |
| Proprietary CT indicators | — | 17 |
| Machine Learning | — | ✅ |
| Market microstructure | — | ✅ |
| Backtesting | ✅ | ✅ |

Learn more at [verida.trade](https://verida.trade) or check the `ct://license/info`
resource inside the app.

---

## Support

- 📧 Email: **contato@verida.trade**
- 🌐 Website: **[verida.trade](https://verida.trade)**
- 🐛 Issues: **[github.com/verida-trade/ct-lab](https://github.com/verida-trade/ct-lab)**

---

## Next Steps

- ➡️ **[01 — CT Lab Desktop](./01-ct-lab-desktop)** — Start by downloading the app
- ➡️ **[02 — AI Provider](./02-provider-ia)** — Configure your preferred AI
- ➡️ **[03 — MCP Connection](./03-conexao-mcp)** — Connect everything
- ➡️ **[04 — First Project](./04-primeiro-projeto)** — Run your first backtest

---

> **Ready to start?** Head to the [first step: CT Lab Desktop →](./01-ct-lab-desktop)

_Last updated: 2026-08-11_
