# Instalação — Guia Completo

Bem-vindo ao **CT Lab**! Esta seção guia você passo a passo desde o download
do aplicativo desktop até a execução do seu primeiro projeto de análise
quantitativa com inteligência artificial.

---

## Pré-requisitos

| Item | Detalhe |
|------|---------|
| Sistema operacional | macOS 11 (Big Sur) ou superior · Ubuntu 20.04+ · Windows 10+ |
| Internet | Necessária para download, ativação e consulta de dados de mercado |
| Conta de provedor de IA | OpenAI, Anthropic, Google AI — ou Ollama rodando localmente |
| Chave de API | Da corretora (Binance) ou provedor de dados (Yahoo Finance), se aplicável |

---

## Roteiro deInstalação

Siga os documentos na ordem abaixo:

| # | Documento | O que você vai fazer |
|---|-----------|----------------------|
| 1 | [CT Lab Desktop](./01-ct-lab-desktop) | Baixar e instalar o aplicativo a partir de verida.trade |
| 2 | [Provedor de IA](./02-provider-ia) | Configurar OpenAI, Anthropic, Google ou Ollama |
| 3 | [Conexão MCP](./03-conexao-mcp) | Conectar o CT Lab Desktop ao ct-mcp-server |
| 4 | [Primeiro Projeto](./04-primeiro-projeto) | Buscar dados, calcular indicadores e rodar um backtest |

> 💡 **Dica**: Se você já é experiente e quer ir direto ao ponto, pule para o
> documento **04 — Primeiro Projeto**. Caso contrário, recomendamos seguir a
> ordem sequencial.

---

## Arquitetura em Resumo

O CT Lab funciona com três componentes principais:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CT LAB DESKTOP (Electron)                                │
│                                                                                  │   │
│   ┌─ Interface de Chat com IA ──┐         ┌─ Configurações & Extensões ─────────┐  │
│   │  • Painel de conversação    │         │  • Provedor de IA (OpenAI, Claude…)  │  │
│   │  • Renderização de gráficos │         │  • Servidor MCP (stdio subprocess)   │  │
│   │  • Resultados de backtest   │         │  • Licença & assinatura premium     │  │
│   └─────────────────────────────┘         └─────────────────────────────────────┘  │
│                        │                                            │       │
│                        ▼                                            ▼       │
│   ┌─────────────────────────────┐         ┌──────────────────────────────────────┐ │
│   │    Provedor de IA (cloud)    │         │        ct-mcp-server (local)          │ │
│   │  OpenAI · Anthropic · Google │         │  • Ferramentas CT (buscar_serie,      │ │
│   │  · Ollama (local)            │         │    rsi, sma, backtest, …)             │ │
│   └─────────────────────────────┘         │  • Cache de séries temporais          │ │
│                                             │  • Indicadores técnicos & CT          │ │
│                                             └──────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

1. **CT Lab Desktop** — aplicativo Electron disponível para macOS, Linux e Windows.
   É por ele que você conversa com a IA, visualiza gráficos e configura tudo.
2. **ct-mcp-server** — processo local (stdio) que acompanha o CT Lab Desktop e
   expõe todas as ferramentas do CT Lab (busca de séries, indicadores técnicos,
   backtests, etc.) para a IA. Não é necessária instalação separada.
3. **Provedor de IA** — serviço que gera as respostas e decide quais ferramentas
   chamar (OpenAI, Anthropic, Google ou Ollama local).

---

## Licenças

O CT Lab oferece dois planos:

| Recurso | Gratuito | Premium |
|---------|----------|---------|
| Cache de séries temporais | 1 série | 100 séries |
| Indicadores técnicos públicos | 36 | 36 |
| Indicadores proprietários CT | — | 17 |
| Machine Learning | — | ✅ |
| Microestrutura de mercado | — | ✅ |
| Backtesting | ✅ | ✅ |

Saiba mais em [verida.trade](https://verida.trade) ou consulte o recurso
`ct://license/info` dentro do aplicativo.

---

## Suporte

- 📧 E-mail: **contato@verida.trade**
- 🌐 Site: **[verida.trade](https://verida.trade)**
- 🐛 Issues: **[github.com/verida-trade/ct-lab](https://github.com/verida-trade/ct-lab)**

---

## Próximos Passos

- ➡️ **[01 — CT Lab Desktop](./01-ct-lab-desktop)** — Comece baixando o aplicativo
- ➡️ **[02 — Provedor de IA](./02-provider-ia)** — Configure sua IA de preferência
- ➡️ **[03 — Conexão MCP](./03-conexao-mcp)** — Conecte tudo
- ➡️ **[04 — Primeiro Projeto](./04-primeiro-projeto)** — Rode seu primeiro backtest

---

> **Pronto para começar?** Siga para o [primeiro passo: CT Lab Desktop →](./01-ct-lab-desktop)

_Última atualização: 2026-08-11_
