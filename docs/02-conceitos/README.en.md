# Concepts & Architecture

> **Folder:** `docs/02-conceitos/`  
> **Target audience:** all levels (beginner → advanced)

This section explains the fundamental concepts of the CT Lab ecosystem — from
 the big-picture overview down to MCP protocol details, the series data model,
 and the difference between **Free** and **Premium** plans.

---

## Table of Contents

| # | File | Topic |
|---|------|-------|
| 1 | [`01-visao-geral`](./01-visao-geral.en.md) | Ecosystem overview — how components connect |
| 2 | [`02-quatro-camadas`](./02-quatro-camadas.en.md) | The 4-layer architecture (Intention, Composition, Consumption, Data) |
| 3 | [`03-uris`](./03-uris.en.md) | The `ct://` URI system — patterns and read templates |
| 4 | [`04-series`](./04-series.en.md) | Series data model (Raw / Derived / Synthetic) |
| 5 | [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) | The MCP protocol in CT: tools, resources, prompts, completions |
| 6 | [`06-free-vs-premium`](./06-free-vs-premium.en.md) | Free vs Premium comparison and how licensing works |

---

## The Architecture on One Page

```
┌──────────── INTENTION & DOCTRINE ─────────────┐
│  goal-first prompts · ct://doutrina/*         │
│  Teaches · suggests method · protects user      │
├──────────── COMPOSITION ───────────────────────┤
│  pipelines · Rhai · compose → ct://derived      │
├──────────── CONSUMPTION ───────────────────────┤
│  strategies (backtest) · features (ML)          │
├──────────── DATA ──────────────────────────────┤
│  series: discover · ingest · repository         │
└───────────────────────────────────────────────┘
```

Each layer has clear responsibilities and depends only on layers below it;
 data access is always controlled by URIs and resource templates.

---

## Reading Prerequisites

| If you are… | Start with |
|-------------|-----------|
| Beginner user | [`01-visao-geral`](./01-visao-geral.en.md) → [`06-free-vs-premium`](./06-free-vs-premium.en.md) |
| Developer integrating MCP | [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) → [`03-uris`](./03-uris.en.md) |
| Quant / data analyst | [`02-quatro-camadas`](./02-quatro-camadas.en.md) → [`04-series`](./04-series.en.md) |

---

## Notation Conventions

| Convention | Meaning |
|------------|---------|
| `ct://series/...` | Resource URI — the AI reads data |
| `buscar_serie` (snake_case) | MCP tool — the AI invokes the action |
| `buscarSerie` (camelCase) | Same function in the TypeScript SDK |
| Free | Available on the free plan |
| Premium | Exclusive to the paid plan |

---

## Next Steps

- **Installation & Setup** → see `docs/01-getting-started/`
- **Indicators** → see `docs/03-indicadores/`
- **Backtest & Strategies** → see `docs/04-backtest/`

---

> **Note:** All communication between the AI and CT Lab components happens via
> MCP (Model Context Protocol). The `ct://` URIs are the universal addressing
> scheme for data, indicators, models, backtests, and doctrine.
