# Conceitos & Arquitetura

> **Pasta:** `docs/02-conceitos/`  
> **Público-alvo:** todos os públicos (iniciante → avançado)

Esta seção explica os conceitos fundamentais do ecossistema CT Lab — da visão
 geral até os detalhes do protocolo MCP, do modelo de dados de séries até a
 diferença entre os planos **Free** e **Premium**.

---

## Sumário

| # | Arquivo | Tópico |
|---|---------|--------|
| 1 | [`01-visao-geral`](./01-visao-geral.pt.md) | Visão geral do ecossistema — como os componentes se conectam |
| 2 | [`02-quatro-camadas`](./02-quatro-camadas.pt.md) | A arquitetura em 4 camadas (Intenção, Composição, Consumo, Dados) |
| 3 | [`03-uris`](./03-uris.pt.md) | O sistema de URIs `ct://` — padrões e templates de leitura |
| 4 | [`04-series`](./04-series.pt.md) | Modelo de dados de séries (Raw / Derived / Synthetic) |
| 5 | [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) | O protocolo MCP no CT: tools, resources, prompts, completions |
| 6 | [`06-free-vs-premium`](./06-free-vs-premium.pt.md) | Comparativo Free vs Premium e como funciona o licensing |

---

## A Arquitetura em uma Página

```
┌──────────── INTENÇÃO & DOUTRINA ──────────────┐
│  prompts goal-first · ct://doutrina/*         │
│  Ensina · sugere método · protege o usuário     │
├──────────── COMPOSIÇÃO ───────────────────────┤
│  pipelines · Rhai · compose → ct://derived      │
├──────────── CONSUMO ──────────────────────────┤
│  estratégias (backtest) · features (ML)         │
├──────────── DADOS ─────────────────────────────┤
│  series: discover · ingest · repository         │
└───────────────────────────────────────────────┘
```

Cada camada tem responsabilidades claras e depende apenas das camadas
inferiores; previews são sempre controlados por URIs e templates de recursos.

---

## Pré-requisitos de Leitura

| Se você é… | Comece por |
|------------|-----------|
| Usuário iniciante | [`01-visao-geral`](./01-visao-geral.pt.md) → [`06-free-vs-premium`](./06-free-vs-premium.pt.md) |
| Desenvolvedor integrando MCP | [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) → [`03-uris`](./03-uris.pt.md) |
| Quant / analista de dados | [`02-quatro-camadas`](./02-quatro-camadas.pt.md) → [`04-series`](./04-series.pt.md) |

---

## Convenções de Notação

| Convenção | Significado |
|-----------|-------------|
| `ct://series/...` | URI de recurso — a IA lê dados |
| `buscar_serie` (snake_case) | Ferramenta MCP — a IA invoca a ação |
| `buscarSerie` (camelCase) | Equivalente na SDK TypeScript |
| <span dir="rtl">Free</span> | Recurso disponível no plano gratuito |
| <span dir="rtl">Premium</span> | Recurso exclusivo do plano pago |

---

## Próximos Passos

- **Instalação & Setup** → veja `docs/01-getting-started/`
- **Indicadores** → veja `docs/03-indicadores/`
- **Backtest & Estratégias** → veja `docs/04-backtest/`

---

> **Nota:** Toda comunicação entre a IA e os componentes do CT Lab acontece via
> MCP (Model Context Protocol). As URIs `ct://` são o endereçamento universal
> para dados, indicadores, modelos, backtests e doutrina.
