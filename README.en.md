# 📈 CT Lab — Public Documentation

> **Crypto/trading analysis platform with AI assistant, technical analysis pipeline, backtesting, machine learning, and microstructure data.**
>
> This repository is a convergence of **documentation**, **executable examples**, and **hands-on recipes** to help you get the most out of CT Lab and the `ct-mcp-server`.

---

## 🧭 About CT Lab

**CT Lab** is a desktop platform for cryptocurrency market analysis that combines:

- 🤖 **AI Assistant** — natural-language interaction via MCP (Model Context Protocol)
- 📊 **Technical indicator pipeline** — 40+ indicators, cascade composition, materialization into time series
- 🔁 **Backtesting** — test Rhai-script strategies with risk metrics and survival testing
- 🧠 **Machine Learning** — ML pipelines, hyperparameter optimization, model application
- 🏗️ **Microstructure** — real-time book, trades, and klines collection (premium feature)
- 📚 **Series repository** — centralized discovery, ingestion, and management of market data

The goal of this repository is to **teach** developers and enthusiasts how to use and extend these capabilities — whether through the desktop app or the command line.

---

## 🚀 Quickstart

### 1. Install CT Lab Desktop

Download the installer at **[verida.trade](https://verida.trade)** (available for Windows, macOS, and Linux).

The `ct-mcp-server` ships bundled with CT Lab Desktop — no separate installation is required. When you open the app, the MCP server starts automatically and all tools become available to the AI assistant.

### 2. Your first command

With the AI assistant open in CT Lab (or your connected MCP client), type:

> *"Fetch BTCUSDT on the 15-minute timeframe and calculate RSI."*

The assistant will fetch the candle series, calculate the RSI indicator, and present the result — all via MCP tools.

---

## 🗺️ Repository Map

```
ct-lab-public/
├── docs/
│   ├── 01-instalacao/         # Installation & setup
│   ├── 02-conceitos/          # Concepts & architecture
│   ├── 03-dados/              # Data: discovery, ingestion, repository
│   ├── 04-indicadores/        # Indicators & pipeline
│   ├── 05-backtest/           # Backtest & strategies
│   ├── 06-ml/                 # Machine Learning
│   ├── 07-microestrutura/     # Microstructure (premium)
│   ├── 08-doutrina/           # Trading doctrine & method
│   ├── 09-prompts/            # MCP prompts & workflows
│   ├── 10-configuracao/       # CT Lab configuration
│   ├── 11-receitas/           # End-to-end recipes
│   ├── 12-faq/                # FAQ & troubleshooting
│   └── 13-api/                # CT Lab REST API (chat, agents, scheduler)
├── examples/
│   ├── rhai/                  # Rhai scripts (strategies + libs)
│   ├── python/               # Python scripts (models + transforms)
│   ├── pipelines/             # Indicator pipeline configs
│   └── esteiras-ml/           # ML pipeline configs
├── README.md                  # Portuguese version (primary)
└── README.en.md               # ← You are here (English)
```

### 📌 Where to start?

| If you want to… | Go to… |
|---|---|
| Install everything from scratch | [`docs/01-instalacao/`](docs/01-instalacao/) |
| Understand the core concepts | [`docs/02-conceitos/`](docs/02-conceitos/) |
| Fetch and manage market data | [`docs/03-dados/`](docs/03-dados/) |
| Build an indicator pipeline | [`docs/04-indicadores/`](docs/04-indicadores/) |
| Test a strategy in backtest | [`docs/05-backtest/`](docs/05-backtest/) |
| Train and apply ML models | [`docs/06-ml/`](docs/06-ml/) |
| Collect microstructure data | [`docs/07-microestrutura/`](docs/07-microestrutura/) |
| Learn the trading method | [`docs/08-doutrina/`](docs/08-doutrina/) |
| Ready-to-use prompts for the assistant | [`docs/09-prompts/`](docs/09-prompts/) |
| Follow a complete step-by-step recipe | [`docs/11-receitas/`](docs/11-receitas/) |
| Troubleshoot a problem | [`docs/12-faq/`](docs/12-faq/) |
| Control CT Lab via API | [`docs/13-api/`](docs/13-api/) |

---

## 📦 Executable Examples

### Rhai — Strategies and Libs

Rhai scripts are used to define backtest strategies and reusable libraries of trading logic.

```rhai
// Example: simple moving average crossover strategy
fn setup() {
    let params = #{
        fast_period: 20,
        slow_period: 50
    };
    return params;
}

fn on_candle(candle, params, state) {
    let fast = sma(candle.close, params.fast_period);
    let slow = sma(candle.close, params.slow_period);

    if fast > slow && !state.in_position {
        return "BUY";
    }
    if fast < slow && state.in_position {
        return "SELL";
    }
    return "HOLD";
}
```

→ See more in [`examples/rhai/`](examples/rhai/)

### Python — Models and Transforms

Python examples use **`uv`** as the package manager and environment. No `pip install` or `conda` is required.

```bash
# Create environment and install dependencies
uv init my-model && cd my-model
uv add pandas numpy scikit-learn
uv run python train.py
```

→ See more in [`examples/python/`](examples/python/)

### Indicator Pipelines and ML Pipelines

JSON/YAML configuration files that define complete pipelines — from raw data to materialized indicators, or from time series to trained models.

→ See more in [`examples/pipelines/`](examples/pipelines/) and [`examples/esteiras-ml/`](examples/esteiras-ml/)

---

## 🖖 Nomenclature: JSON/RPC vs TypeScript SDK

CT Lab tools can be invoked in two ways, depending on the context:

| Context | Convention | Example |
|---|---|---|
| **MCP (JSON/RPC)** | `snake_case` | `buscar_serie`, `calcular_rsi`, `ct_backtest` |
| **TypeScript SDK** | `camelCase` | `buscarSerie`, `calcularRsi`, `ctBacktest` |

Throughout all examples in this repository, the correct syntax is used for each context.

---

## 🔧 Requirements

- **CT Lab Desktop** — latest version ([verida.trade](https://verida.trade))
- **Python ≥ 3.11** with **`uv`** ([astral.sh/uv](https://docs.astral.sh/uv/)) — only for Python examples

---

## 📜 License

This repository uses a dual license for **code examples**:

- **MIT License** OR **Apache License 2.0** — your choice.

Files in `examples/` (Rhai scripts, Python scripts, pipeline configs, and ML pipeline configs) are covered by this dual license. You may use, modify, and redistribute them under the terms of either license.

The **documentation** in `docs/` is © **Verida Trade**. All rights reserved. The documentation is provided for educational and reference purposes, but may not be copied, redistributed, or published in other repositories without express permission.

> ⚠️ **This repository does not accept community Pull Requests.** It is a locked repository. To report bugs, please use [GitHub Issues](https://github.com/verida-trade/ct-lab/issues).

---

## 🔗 Links

| Resource | URL |
|---|---|
| 🌐 Product site | [verida.trade](https://verida.trade) |
| 📁 Repository (this one) | [github.com/verida-trade/ct-lab](https://github.com/verida-trade/ct-lab) |
| 🐛 Bug reports | [GitHub Issues](https://github.com/verida-trade/ct-lab/issues) |
| 📦 MCP server | [github.com/alexandrelima-starkbank/ct-mcp](https://github.com/alexandrelima-starkbank/ct-mcp) |

---

## 🤝 Contributing

While we do not accept Pull Requests, we greatly value your contributions in the form of:

- 🐛 **Bug reports** — open an [Issue](https://github.com/verida-trade/ct-lab/issues) with a detailed description, reproduction steps, and relevant logs
- 💡 **Content suggestions** — propose new tutorials or recipes via Issues
- 📣 **Feedback** — tell us how you're using CT Lab and what could be improved

---

<p align="center">
  <strong>CT Lab</strong> · Made with 🤍 by <a href="https://verida.trade">Verida Trade</a>
  <br>
  Agentic AI Foundation — AAIF
</p>
