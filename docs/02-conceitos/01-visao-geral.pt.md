# Visão Geral do Ecossistema CT Lab

> **Pasta:** `docs/02-conceitos/01-visao-geral.pt.md`  
> **Leitura relacionada:** [`02-quatro-camadas`](./02-quatro-camadas.pt.md) ·
> [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md)  
> **Público-alvo:** iniciantes e integradores

---

## O que é o CT Lab?

O CT Lab é uma plataforma de análise quantitativa e trading algorítmico que
opera localmente na máquina do usuário. Ele expõe um conjunto de ferramentas
para descoberta, ingestão, transformação e backtest de séries financeiras —
tudo acessível a um LLM (Large Language Model) via **MCP** (Model Context
Protocol).

A premissa central: o usuário conversa naturalmente com a IA, e a IA usa
ferramentas MCP para ler dados, calcular indicadores, rodar backtests e montar
pipelines — sem que o usuário precise escrever código.

---

## Diagrama de Componentes

```
 ┌───────────────────────────────────────────────────────────────┐
 │                       Usuário                                 │
 │               (conversa em linguagem natural)                 │
 └───────────────────────┬───────────────────────────────────────┘
                         │  "Qual a SMA 20 do BTCUSDT em 1h?"
                         ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                  CT Lab Desktop (Electron UI)                  │
 │  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐  │
 │  │   Chat UI    │    │  Gráficos /   │    │  Gestão deConta  │  │
 │  │   (Markdown)  │    │  Tabelas      │    │  Free / Premium  │  │
 │  └──────┬───────┘    └──────────────┘    └──────────────────┘  │
 └─────────┼─────────────────────────────────────────────────────┘
           │  MCP stdio (JSON-RPC)
           ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                    ct-mcp-server                               │
 │                                                               │
 │   Tools (ações: buscar_serie, sma, ct_backtest, …)             │
 │   Resources (leituras via templates: tail/head/sample)        │
 │   Prompts (workflows guiados: saudacao, backtest, …)           │
 │   Completions (autocompletar argumentos de prompts)            │
 └─────────┬─────────────────────────────────────────────────────┘
           │  HTTP (localhost)
           ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                    ct-labd (servidor HTTP local)               │
 │                                                               │
 │   ┌──────────┐   ┌───────────┐   ┌───────────┐   ┌────────┐  │
 │   │ Série    │   │Indicadores│   │ Backtest  │   │  ML    │  │
 │   │ Repo     │   │ Engine    │   │ Engine    │   │Engine  │  │
 │   └──────────┘   └───────────┘   └───────────┘   └────────┘  │
 │                                                               │
 │   Cache (até 1 séries Free · até 100 Premium)                  │
 │   Providers: Binance, Yahoo Finance, CSV, …                    │
 └───────────────────────────────────────────────────────────────┘
           │
           ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                       LLM Provider                             │
 │     (OpenAI / Anthropic / Google / Local)                      │
 │     Recebe tool definitions, invoca tools, raciocina           │
 └───────────────────────────────────────────────────────────────┘
```

---

## Fluxo de Dados — Passo a Passo

1. **O usuário faz uma pergunta** em linguagem natural no Chat UI.
2. **O LLM recebe** a pergunta, o histórico e as definições de ferramentas MCP
   expostas pelo `ct-mcp-server`.
3. **O LLM decide** se precisa chamar uma ferramenta — por exemplo,
   `buscar_serie` para localizar a série `ct://series/binance/BTCUSDT/1h`.
4. **O `ct-mcp-server`** encaminha a chamada via HTTP para o `ct-labd`,
   que buscar no cache ou consulta o provider (Binance, Yahoo, etc.).
5. **A ferramenta retorna** uma URI + metadados — nunca as linhas de dados
   cruas diretamente.
6. **O LLM lê os dados** via recurso (resource template), por exemplo
   `ct://series/binance/BTCUSDT/1h/tail/20` para as últimas 20 linhas.
7. **O LLM calcula** indicadores adicionais se necessário (ex: `sma`), e
   sintetiza uma resposta em linguagem natural.
8. **O usuário vê** a resposta renderizada no Chat UI, com gráficos e
   tabelas opcionalmente.

---

## Componentes em Resumo

| Componente | Tecnologia | Função |
|------------|------------|--------|
| **CT Lab Desktop** | Electron + UI Web | Interface gráfica do usuário (chat, gráficos, gestão de conta) |
| **ct-mcp-server** | MCP stdio (JSON-RPC) | Ponte MCP entre LLM e o backend local |
| **ct-labd** | Servidor HTTP (localhost) | Motor de dados, indicadores, backtest e ML |
| **LLM Provider** | OpenAI / Anthropic / etc. | Raciocínio e invocação de ferramentas |
| **Providers de dados** | Binance, Yahoo, CSV | Fontes externas de séries OHLCV |

---

## Por que MCP?

O MCP (Model Context Protocol) é um padrão aberto que permite que LLMs
descubram e invoquem ferramentas externas de forma estruturada. No CT Lab, o
MCP resolve três problemas:

| Problema | Solução MCP |
|----------|-------------|
| Como a IA encontra dados? | **Resources** — URIs `ct://` com templates `tail/head` |
| Como a IA executa ações? | **Tools** — funções MCP que retornam URI + metadados |
| Como o usuário inicia workflows? | **Prompts** — templates guiados invocados pelo usuário |

Veja [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) para o detalhamento completo.

---

## Instalação e Primeira Execução

```bash
# Clone o repositório público
git clone https://github.com/verida-trade/ct-lab.git
cd ct-lab

# Instale dependências (Node.js ≥ 20)
npm install

# Inicie o CT Lab Desktop
npm start
```

Após iniciar, o CT Lab Desktop:
1. Sobe o `ct-labd` na porta local (ex: `http://localhost:8420`).
2. Sobe o `ct-mcp-server` em modo stdio.
3. Conecta-se ao LLM configurado (chave de API via variável de ambiente).

---

## Verificação Rápida

Após a primeira execução, teste com uma pergunta simples no chat:

```
  Usuário: Qual o preço atual do BTCUSDT na Binance?

  IA: → chama buscar_serie(provider="binance", symbol="BTCUSDT", timeframe="1m")
     → recebe ct://series/binance/BTCUSDT/1m
     → lê ct://series/binance/BTCUSDT/1m/tail/1
     → responde com o preço atual
```

Se você vê a URI retornada e o preço na resposta, o ecossistema está
funcionando corretamente.

---

## Próximos Passos

- [`02-quatro-camadas`](./02-quatro-camadas.pt.md) — detalhe das 4 camadas
- [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) — como o MCP funciona no CT
- [`06-free-vs-premium`](./06-free-vs-premium.pt.md) — planos e licenciamento
