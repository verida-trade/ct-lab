# O Protocolo MCP no CT Lab

> **Pasta:** `docs/02-conceitos/05-mcp-protocolo.pt.md`  
> **Leitura relacionada:** [`03-uris`](./03-uris.pt.md) ·
> [`01-visao-geral`](./01-visao-geral.pt.md)  
> **Público-alvo:** desenvolvedores e integradores

---

## O que é o MCP?

O **MCP** (Model Context Protocol) é um padrão aberto que permite que LLMs
descubram e invoquem ferramentas, leiam recursos e apresentem prompts guiados
ao usuário. No CT Lab, o MCP é o protocolo único de comunicação entre a IA e o
backend local (`ct-labd`).

O `ct-mcp-server` funciona em modo **stdio** (JSON-RPC sobre stdin/stdout),
fazendo ponte entre o LLM e o `ct-labd`.

---

## Os 4 Primitivos do MCP

O MCP define quatro primitivos. No CT Lab, cada um tem um papel específico:

```
┌──────────────────────────────────────────────────────────────┐
│                     MCP no CT Lab                             │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────┐ │
│  │   Tools    │  │ Resources  │  │  Prompts   │  │Complet.│ │
│  │ (ações)    │  │ (leituras) │  │ (workflows)│  │(autocomp)│
│  │ snake_case │  │ ct:// URIs │  │ user invoke│  │ args   │ │
│  └────────────┘  └────────────┘  └────────────┘  └────────┘ │
│       │              │                │                │      │
│       ▼              ▼                ▼                ▼      │
│  Retorna URI    Retorna dados    Retorna plano    Sugestões  │
│  + metadados    (linhas reais)   estruturado       de args   │
└──────────────────────────────────────────────────────────────┘
```

| Primitivo | Quem invoca | O que retorna | Exemplo |
|-----------|-------------|----------------|---------|
| **Tool** | A IA (modelo) | URI + metadados | `buscar_serie` → `ct://series/...` |
| **Resource** | A IA (modelo) lê | Dados reais (linhas) | `ct://series/.../tail/20` |
| **Prompt** | O **usuário** invoca | Workflow estruturado | `backtest`, `saudacao` |
| **Completion** | O cliente UI | Sugestões de argumentos | Autocompletar `provider` |

---

## 1. Tools (Ferramentas)

Tools são **ações** que a IA invoca para descobrir, criar ou transformar
recursos. O ponto crítico: **tools nunca retornam linhas de dados cruas**.
Elas retornam uma **URI + metadados**.

### Convenção de nomenclatura

| Ambiente | Notação | Exemplo |
|----------|---------|---------|
| MCP (protocolo direto) | `snake_case` | `buscar_serie`, `ct_backtest` |
| SDK TypeScript | `camelCase` | `buscarSerie`, `ctBacktest` |

### Exemplos de Tools

```python
# MCP: a IA invoca buscar_serie (snake_case)
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import buscar_serie; "
    "uri = buscar_serie(provider='binance', symbol='BTCUSDT', timeframe='1h'); "
    "print(uri)"
], capture_output=True, text=True)
print(result.stdout)
# → ct://series/binance/BTCUSDT/1h
```

```typescript
// SDK TypeScript: camelCase
const uri = await Ct.buscarSerie({
    provider: "binance",
    symbol: "BTCUSDT",
    timeframe: "1h",
});
// → { uri: "ct://series/binance/BTCUSDT/1h", meta: { ... } }
```

### Categorias de Tools

| Categoria | Exemplos | Plano |
|-----------|---------|-------|
| **Dados** | `buscar_serie`, `importar_csv`, `listar_series` | Free |
| **Indicadores públicos** (36) | `sma`, `ema`, `rsi`, `macd`, `bollinger`, … | Free |
| **Indicadores CT** (17) | `ct_bop`, `ct_tfi`, `ct_bfi`, `ct_obi`, … | Premium |
| **Backtest** | `ct_backtest`, `ct_buscar_backtests` | Free |
| **ML** | `montar_esteira_ml`, `aplicar_modelo`, `otimizar_hiperparametros` | Premium |
| **Microestrutura** | `coletar_book`, `coletar_trades`, `consultar_book` | Premium |
| **Libs** | `salvar_lib`, `ler_lib`, `listar_libs` | Premium |
| **Coleta** | `criar_tarefa_coleta`, `parar_coleta`, `listar_tarefas` | Free |
| **Sobrevivência** | `ct_testar_sobrevivencia` | Premium |
| **Billing** | `comprar_premium`, `cancelar_assinatura` | Free |

---

## 2. Resources (Recursos)

Resources são **leituras de dados**. A IA lê o conteúdo real de um recurso
usando sua URI. No CT Lab, os resources seguem templates que permitem ler
fatias da série sem transferir tudo.

### Templates de leitura

| Template | Sintaxe | Limit |
|----------|---------|-------|
| `tail` | `ct://series/.../<tf>/tail/<n>` | n ≤ 200 |
| `head` | `ct://series/.../<tf>/head/<n>` | n ≤ 200 |
| `sample` | `ct://series/.../<tf>/sample/<n>` | n ≤ 200 |

### Exemplo: Lendo dados via resource

```python
# A IA encontrou a série via tool, agora LÊ os dados via resource
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "data = read_resource('ct://series/binance/BTCUSDT/1h/tail/10'); "
    "print(data.timestamps[:3]); "
    "print(data.columns['close'][:3])"
], capture_output=True, text=True)
print(result.stdout)
# [1700000000, 1700003600, 1700007200]
# [67100.5, 67150.2, 67200.8]
```

### Catálogos como resources

```python
# Catálogo de indicadores é um resource vivo
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "catalog = read_resource('ct://indicators/catalog'); "
    "print(catalog)"
], capture_output=True, text=True)
print(result.stdout)
# [{ name: 'sma', category: 'public' }, { name: 'rsi', ... }, ...]
```

---

## 3. Prompts (Workflows Guiados)

Prompts são **workflows estruturados** que o **usuário** invoca — não a IA.
Eles guiam a conversa com instruções, parâmetros sugeridos e validações.

### Prompts Públicos (Free)

| Prompt | Função |
|--------|--------|
| `saudacao` | Saudação inicial e onboarding |
| `comecar` | Guia de início rápido |
| `backtest` | Estrutura uma requisição de backtest passo a passo |

### Prompts Premium

| Prompt | Função |
|--------|--------|
| `coleta` | Configura tarefas de coleta de dados |
| `esteira` | Monta esteira de ML completa |
| `fundacao` | Cria fundação de dados para projeto |
| `regime` | Análise de regime de mercado |

### Como o usuário invoca um prompt

```
  Usuário (no chat UI): /backtest
  
  → O prompt 'backtest' é injetado no contexto do LLM
  → O LLM recebe instruções estruturadas:
    "Pergunte qual ativo, timeframe, estratégia, parâmetros…"
  → O LLM conduz o diálogo guiado com o usuário
  → Ao final, chama ct_backtest com os parâmetros coletados
```

---

## 4. Completions (Autocompletar)

Completions fornecem **sugestões de argumentos** para prompts. Por exemplo,
quando o usuário começa a digitar o argumento `provider` no prompt `backtest`,
o completion sugere: `binance`, `yahoo`, `csv`.

```
  Usuário digita: /backtest provider=bin┄
                                   ▼
  Completion retorna: ["binance"]
  
  Usuário seleciona: binance
```

> Completions rodam no cliente UI e não enviam dados ao LLM — são purely
> client-side suggestion queries ao `ct-mcp-server`.

---

## O que o Modelo vê vs. O que o Usuário invoca

| Primitivo | Quem inicia | O modelo LLM vê? |
|-----------|-------------|-------------------|
| Tool | A IA decide invocar | ✅ Sim — no seu contexto de reasoning |
| Resource | A IA decide ler | ✅ Sim — o conteúdo entra no contexto |
| Prompt | O usuário invoca | ✅ Sim — injetado como instrução no contexto |
| Completion | O cliente UI solicita | ❌ Não — processado client-side |

### Fluxo típico

```
 1. Usuário:     /backtest
 2. Prompt:      Injetado no contexto do LLM
 3. LLM pergunta: "Qual ativo?"        →Usuário: BTCUSDT
 4. LLM pergunta: "Qual timeframe?"    →Usuário: 1h
 5. LLM decide:  Invocar tool buscar_serie
 6. Tool:        Retorna ct://series/binance/BTCUSDT/1h
 7. LLM decide:  Ler resource tail/100
 8. Resource:    Retorna 100 linas OHLCV
 9. LLM decide:  Invocar tool sma(period=20)
10. Tool:        Retorna ct://derived/sma_...
11. LLM decide:  Invocar tool ct_backtest(...)
12. Tool:        Retorna ct://backtest/a1b2c3
13. LLM:         Apresenta resultado ao usuário
```

---

## Próximos Passos

- [`03-uris`](./03-uris.pt.md) — todos os padrões de URI
- [`04-series`](./04-series.pt.md) — modelo de dados
- [`06-free-vs-premium`](./06-free-vs-premium.pt.md) — gates de licença
