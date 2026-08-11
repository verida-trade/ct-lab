# 📈 CT Lab — Documentação Pública

> **Plataforma de análise de criptomoedas e trading com assistente de IA, pipeline de análise técnica, backtesting, machine learning e dados de microestrutura.**
>
> Este repositório convergence de **documentação**, **exemplos executáveis** e **receitas práticas** para você extrair o máximo do CT Lab e do `ct-mcp-server`.

---

## 🧭 Sobre o CT Lab

O **CT Lab** é uma plataforma desktop para análise de mercados de criptomoedas que combina:

- 🤖 **Assistente de IA** — interação em linguagem natural via MCP (Model Context Protocol)
- 📊 **Pipeline de indicadores técnicos** — mais de 40 indicadores, composição em cascata, materialização em série temporal
- 🔁 **Backtesting** — teste de estratégias em scripts Rhai, com métricas de risco e sobrevivência
- 🧠 **Machine Learning** — esteiras ML, otimização de hiperparâmetros, aplicação de modelos
- 🏗️ **Microestrutura** — coleta de book, trades e klines em tempo real (funcionalidade premium)
- 📚 **Repositório de séries** — descoberta, ingestão e gestão centralizada de dados de mercado

O objetivo deste repositório é **ensinar** desenvolvedores e entusiastas a usar e estender essas capacidades — seja pelo app desktop ou pela linha de comando.

---

## 🚀 Quickstart

### 1. Instale o CT Lab Desktop

Baixe o instalador em **[verida.trade](https://verida.trade)** (disponível para Windows, macOS e Linux).

O `ct-mcp-server` acompanha o CT Lab Desktop — não é necessária instalação separada. Ao abrir o aplicativo, o servidor MCP é iniciado automaticamente e todas as ferramentas ficam disponíveis para o assistente de IA.

### 2. Seu primeiro comando

Com o assistente de IA aberto no CT Lab (ou no seu cliente MCP conectado), digite:

> *"Busque BTCUSDT no timeframe de 15 minutos e calcule o RSI."*

O assistente vai buscar a série de candles, calcular o indicador RSI e apresentar o resultado — tudo via ferramentas MCP.

---

## 🗺️ Mapa do Repositório

```
ct-lab-public/
├── docs/
│   ├── 01-instalacao/         # Instalação & configuração inicial
│   ├── 02-conceitos/          # Conceitos & arquitetura
│   ├── 03-dados/              # Dados: descoberta, ingestão, repositório
│   ├── 04-indicadores/        # Indicadores & pipeline de indicadores
│   ├── 05-backtest/           # Backtest & estratégias
│   ├── 06-ml/                 # Machine Learning
│   ├── 07-microestrutura/     # Microestrutura (premium)
│   ├── 08-doutrina/           # Doutrina de trading & método
│   ├── 09-prompts/            # Prompts MCP & workflows
│   ├── 10-configuracao/       # Configuração do CT Lab
│   ├── 11-receitas/           # Receitas end-to-end
│   ├── 12-faq/                # FAQ & troubleshooting
│   └── 13-api/                # API REST do CT Lab (chat, agentes, scheduler)
├── examples/
│   ├── rhai/                  # Scripts Rhai (estratégias + libs)
│   ├── python/              # Scripts Python (modelos + transforms)
│   ├── pipelines/             # Configs de pipeline de indicadores
│   └── esteiras-ml/           # Configs de esteira ML
├── README.md                  # ← Você está aqui (Português)
└── README.en.md               # English version
```

### 📌 Onde começar?

| Se você quer… | Vá para… |
|---|---|
| Instalar tudo do zero | [`docs/01-instalacao/`](docs/01-instalacao/) |
| Entender os conceitos básicos | [`docs/02-conceitos/`](docs/02-conceitos/) |
| Buscar e gerenciar dados de mercado | [`docs/03-dados/`](docs/03-dados/) |
| Criar um pipeline de indicadores | [`docs/04-indicadores/`](docs/04-indicadores/) |
| Testar uma estratégia em backtest | [`docs/05-backtest/`](docs/05-backtest/) |
| Treinar e aplicar modelos de ML | [`docs/06-ml/`](docs/06-ml/) |
| Coletar dados de microestrutura | [`docs/07-microestrutura/`](docs/07-microestrutura/) |
| Aprender o método de trading | [`docs/08-doutrina/`](docs/08-doutrina/) |
| Prompts prontos para o assistente | [`docs/09-prompts/`](docs/09-prompts/) |
| Seguir uma receita completa passo a passo | [`docs/11-receitas/`](docs/11-receitas/) |
| Resolver um problema | [`docs/12-faq/`](docs/12-faq/) |
| Controlar o CT Lab por API | [`docs/13-api/`](docs/13-api/) |

---

## 📦 Exemplos Executáveis

### Rhai — Estratégias e Libs

Scripts Rhai são usados para definir estratégias de backtest e bibliotecas reutilizáveis de lógica de trading.

```rhai
// Exemplo: estratégia simples de cruzamento de média móvel
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

→ Veja mais em [`examples/rhai/`](examples/rhai/)

### Python — Modelos e Transforms

Os exemplos Python usam **`uv`** como gerenciador de pacotes e ambiente. Nenhum `pip install` ou `conda` é necessário.

```bash
# Criar ambiente e instalar dependências
uv init my-model && cd my-model
uv add pandas numpy scikit-learn
uv run python train.py
```

→ Veja mais em [`examples/python/`](examples/python/)

### Pipelines de Indicadores e Esteiras ML

Aquivos de configuração JSON/YAML que definem pipelines completos — do dado bruto ao indicador materializado, ou da série temporal ao modelo treinado.

→ Veja mais em [`examples/pipelines/`](examples/pipelines/) e [`examples/esteiras-ml/`](examples/esteiras-ml/)

---

## 🖖 Nomenclatura: JSON/RPC vs TypeScript SDK

As ferramentas do CT Lab podem ser chamadas de duas formas, dependendo do contexto:

| Contexto | Convenção | Exemplo |
|---|---|---|
| **MCP (JSON/RPC)** | `snake_case` | `buscar_serie`, `calcular_rsi`, `ct_backtest` |
| **TypeScript SDK** | `camelCase` | `buscarSerie`, `calcularRsi`, `ctBacktest` |

Em todos os exemplos deste repositório, a sintaxe correta é usada conforme o contexto.

---

## 🔧 Requisitos

- **CT Lab Desktop** — versão mais recente ([verida.trade](https://verida.trade))
- **Python ≥ 3.11** com **`uv`** ([astral.sh/uv](https://docs.astral.sh/uv/)) — apenas para exemplos Python

---

## 📜 Licença

Este repositório adota uma licença dupla para os **exemplos de código**:

- **MIT License** OU **Apache License 2.0** — à sua escolha.

Os arquivos em `examples/` (scripts Rhai, Python, configs de pipeline e esteiras ML) estão sob esta licença dupla. Você pode usá-los, modificá-los e redistribuí-los conforme os termos de qualquer uma das duas licenças.

A **documentação** em `docs/` é © **Verida Trade**. Todos os direitos reservados. A documentação é fornecida para uso educacional e de consulta, mas não pode ser copiada, redistribuída ou publicada em outros repositórios sem autorização expressa.

> ⚠️ **Este repositório não aceita Pull Requests da comunidade.** É um repositório fechado (locked). Para reportar erros, use as [Issues do GitHub](https://github.com/verida-trade/ct-lab/issues).

---

## 🔗 Links

| Recurso | URL |
|---|---|
| 🌐 Site do produto | [verida.trade](https://verida.trade) |
| 📁 Repositório (este) | [github.com/verida-trade/ct-lab](https://github.com/verida-trade/ct-lab) |
| 🐛 Reportar bugs | [GitHub Issues](https://github.com/verida-trade/ct-lab/issues) |
| 📦 Servidor MCP | [github.com/alexandrelima-starkbank/ct-mcp](https://github.com/alexandrelima-starkbank/ct-mcp) |

---

## 🤝 Contribuindo

Embora não aceitemos Pull Requests, valorizamos muito sua contribuição na forma de:

- 🐛 **Bug reports** — abra uma [Issue](https://github.com/verida-trade/ct-lab/issues) com descrição detalhada, passos para reproduzir e logs relevantes
- 💡 **Sugestões de conteúdo** — proponha novos tutoriais ou receitas via Issues
- 📣 **Feedback** — conte-nos como está usando o CT Lab e o que poderia ser melhor

---

<p align="center">
  <strong>CT Lab</strong> · Feito com 🤍 pela <a href="https://verida.trade">Verida Trade</a>
  <br>
  Agentic AI Foundation — AAIF
</p>
