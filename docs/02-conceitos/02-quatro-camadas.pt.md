# A Arquitetura em 4 Camadas

> **Pasta:** `docs/02-conceitos/02-quatro-camadas.pt.md`  
> **Leitura relacionada:** [`01-visao-geral`](./01-visao-geral.pt.md) ·
> [`03-uris`](./03-uris.pt.md) · [`04-series`](./04-series.pt.md)  
> **Público-alvo:** desenvolvedores e quants

---

## Visão Geral

O CT Lab é estruturado em **4 camadas verticais**, cada uma com
responsabilidades distintas. As camadas superiores dependem das inferiores,
mas nunca o contrário — os dados fluem de baixo para cima, e a intenção flui
de cima para baixo.

```
┌─────────────────────────────────────────────────────────────────┐
│  CAMADA 1 — INTENÇÃO & DOUTRINA                                  │
│  Do objetivo do usuário → ensina · sugere método · protege       │
│  prompts goal-first · ct://doutrina/* · blueprints               │
├─────────────────────────────────────────────────────────────────┤
│  CAMADA 2 — COMPOSIÇÃO                                            │
│  Constrói SINAIS e FEATURES                                       │
│  pipelines · scripts Rhai · compose → ct://derived               │
├─────────────────────────────────────────────────────────────────┤
│  CAMADA 3 — CONSUMO                                               │
│  Estratégias (backtest) e features (ML)                            │
│  ct_backtest · montar_esteira_ml · aplicar_modelo                 │
├─────────────────────────────────────────────────────────────────┤
│  CAMADA 4 — DADOS                                                 │
│  Séries: descoberta · ingestão · repositório                      │
│  buscar_serie · importar_csv · ct://series/*                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Camada 1 — Intenção & Doutrina

### Responsabilidade

Traduzir o objetivo do usuário em um plano de ação seguro e estruturado. É
a camada mais alta — ela **ensina** o usuário sobre o método, **sugere** o
caminho analítico adequado e **protege** contra decisões perigosas
(overfitting, viés de lookahead, etc.).

### Componentes

| Componente | Descrição |
|------------|-----------|
| **Prompts** | Workflows guiados que o usuário invoca (ex: `saudacao`, `backtest`, `comecar`) |
| **Doutrina** | Princípios de trading acessíveis via `ct://doutrina/*` |
| **Blueprints** | Modelos de estratégia pré-empacotados |

### Exemplo concreto

O usuário diz: *"Quero testar uma estratégia de cruzamento de médias no
BTCUSDT."*

A camada de Intenção (via prompt `backtest`) reconhece o objetivo, sugere os
parâmetros razoáveis (janela de SMA, período de backtest), alerta sobre pitfalls
comuns (overfitting em dados in-sample) e estrutura a requisição para a camada
de Consumo executar o backtest.

```python
# O prompt guiado estruturou a requisição — resultado executado via Python
import subprocess
result = subprocess.run(
    ["uv", "run", "python", "-c",
     "from ct_lab import executar_backtest; "
     "executar_backtest(strategy='sma_cross', symbol='BTCUSDT', timeframe='1h')"],
    capture_output=True, text=True
)
print(result.stdout)
```

> **Nota:** Na prática, a IA chama a ferramenta MCP `ct_backtest` diretamente.
> O exemplo em Python acima ilustra como seria uma execução equivalente via SDK.

---

## Camada 2 — Composição

### Responsabilidade

Transformar dados brutos em **sinais** e **features**. Esta camada combina
indicadores, operações matemáticas e lógica condicional para produzir séries
derivadas e sintéticas.

### Componentes

| Componente | Descrição |
|------------|-----------|
| **Pipelines** | Sequências de operações (`montar_pipeline_indicadores`) |
| **Scripts Rhai** | Linguagem de expressão para transformações inline |
| **Compose** | Combina N séries em uma série sintética (`compor_serie`) |
| **Materializar** | Persiste um indicador como série derivada (`materializar_indicador`) |

### Exemplo concreto

```python
# Monta um pipeline: SMA(20) + RSI(14) → série derivada "momentum_signal"
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import montar_pipeline; "
    "montar_pipeline("
    "  source='ct://series/binance/BTCUSDT/1h',"
    "  ops=['sma:20', 'rsi:14'],"
    "  name='momentum_signal'"
    ")"
], capture_output=True, text=True)
print(result.stdout)
# → ct://derived/momentum_signal
```

A partir desse ponto, a série `ct://derived/momentum_signal` está disponível
para consumo pelo backtest ou ML.

---

## Camada 3 — Consumo

### Responsabilidade

Consumir as séries (brutas, derivadas ou sintéticas) para dois fins:
**estratégias de trading** (backtest) e **features de ML** (treino/predição).

### Componentes

| Componente | Descrição |
|------------|-----------|
| **Backtest Engine** | `ct_backtest` — simula estratégias com regras de entrada/saída |
| **ML Pipeline** | `montar_esteira_ml`, `otimizar_hiperparametros`, `aplicar_modelo` |
| **Otimização** | Busca de hiperparâmetros sobre séries |
| **Teste de Sobrevivência** | `ct_testar_sobrevivencia` (Premium) — robustez estatística |

### Exemplo concreto

```python
# Roda um backtest sobre a série derivada criada na Camada 2
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import ct_backtest; "
    "ct_backtest("
    "  series='ct://derived/momentum_signal',"
    "  strategy='sma_cross',"
    "  params={'fast': 20, 'slow': 50}"
    ")"
], capture_output=True, text=True)
print(result.stdout)
# → ct://backtest/a1b2c3
```

O resultado é uma URI de backtest que pode ser lida via
`ct://backtest/a1b2c3`.

---

## Camada 4 — Dados

### Responsabilidade

Descobrir, ingerir e armazenar séries financeiras. Esta é a fundação — sem
dados, nenhuma camada acima funciona.

### Componentes

| Componente | Descrição |
|------------|-----------|
| **Descoberta** | `buscar_serie`, `listar_series`, `top_ativos` |
| **Ingestão** | `importar_csv`, `buscar_binance`, `buscar_yahoo`, `criar_tarefa_coleta` |
| **Repositório** | Cache local de séries (`ct://series/*`) |
| **Providers** | Binance, Yahoo Finance, CSV, e mais |

### Exemplo concreto

```python
# Descobre e ingere a série de 1h do BTCUSDT da Binance
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import buscar_serie; "
    "buscar_serie(provider='binance', symbol='BTCUSDT', timeframe='1h')"
], capture_output=True, text=True)
print(result.stdout)
# → ct://series/binance/BTCUSDT/1h
```

A URI `ct://series/binance/BTCUSDT/1h` passa a existir no repositório e pode
ser lida por todas as camadas superiores.

---

## Fluxo Completo: Das 4 Camadas

```
 Usuário: "Teste SMA cross no Bitcoin 1h com custom RSI"
    │
    ▼
 [Camada 1] Prompt 'backtest' → estrutura plano de ação
    │
    ▼
 [Camada 4] buscar_serie → ct://series/binance/BTCUSDT/1h
    │                                     │
    ▼                                     ▼
 [Camada 2] sma(20) + rsi(14) → ct://derived/my_signal
    │
    ▼
 [Camada 3] ct_backtest(strategy='sma_cross') → ct://backtest/abc123
    │
    ▼
    IA apresenta resultado ao usuário (com gráficos)
```

---

## Princípios de Design

| Princípio | Aplicação |
|-----------|-----------|
| **Separação de responsabilidades** | Cada camada tem um papel claro |
| **Endereçamento universal via URI** | Toda série, indicador, modelo e backtest tem uma URI `ct://` |
| **Nunca retornar linhas cruas da tool** | Tools retornam URI + meta; dados lidos via recursos |
| **Fluxo unidirecional** | Dados sobem (Dados→Consumo); intenção desce (Intenção→Composição) |
| **Cache transiente** | Séries podem ser materializadas mas o repositório é transitório |

---

## Próximos Passos

- [`03-uris`](./03-uris.pt.md) — como endereçar qualquer recurso
- [`04-series`](./04-series.pt.md) — detalhe do modelo de séries
- [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) — como a IA interage
