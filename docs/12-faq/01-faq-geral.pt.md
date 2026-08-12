# FAQ Geral

> Perguntas frequentes de quem está começando com o CT Lab.

## Preciso de qual provedor de IA?

OpenAI gpt-4o ou Anthropic claude-sonnet-4-20250514 — ambos têm excelente suporte ao MCP. Ollama funciona localmente (offline) mas pode ter limitações com function calling. Google Gemini também é suportado.

| Provedor | Requer internet | Function calling | Recomendado |
|---|---|---|---|
| OpenAI (gpt-4o) | ✅ | ✅ Excelente | ⭐ Iniciantes |
| Anthropic (claude-sonnet) | ✅ | ✅ Excelente | ⭐ Iniciantes |
| Google (gemini-2.0-flash) | ✅ | ✅ Bom | |
| Ollama (llama3, mistral) | ❌ | ⚠️ Varia por modelo | Privacidade total |

## Posso usar offline?

Parcialmente. O Ollama roda 100% local (sem nuvem), mas você precisa de internet para buscar dados de mercado (candles, klines). O cache de séries é local — uma vez que os dados estão em cache, o backtest e os indicadores funcionam offline.

## Qual a diferença entre `auto` e `chat`?

| Modo | Comportamento | Quando usar |
|---|---|---|
| `auto` | Agente: decide, chama tools, executa | Análise, backtest, pipeline |
| `chat` | Conversa direta: só responde, não executa tools | Perguntas conceituais, brainstorm |

Para usar o ct-mcp-server, **sempre use `auto`**. Em `chat` a IA não consegue invocar ferramentas.

## Quantas ferramentas (tools) o ct-mcp-server oferece?

~120 tools MCP (varia conforme plano e variáveis de ambiente):

| Categoria | Quantidade | Plano |
|---|---|---|
| Indicadores públicos | 36 | Free |
| Indicadores CT proprietários | 17 | Premium |
| Dados (descoberta, ingestão) | ~15 | Free |
| Backtest | 2 | Free |
| ML pipeline | ~20 | Premium |
| Microestrutura | ~10 | Free/Premium |
| Libs, filtros, prompts | ~10 | Free/Premium |
| Billing, config | ~5 | Free |

> O número exato muda conforme versões. Para ver a lista completa: pergunte à IA "liste todas as ferramentas disponíveis" ou use `tools/list` no protocolo MCP.

## O que significa "1 série" no plano Free?

No plano Free, o cache de séries comporta **1 série**. Isso significa:

- ✅ Buscar `BTCUSDT 15m` → 1 série em cache
- ❌ Buscar `ETHUSDT 15m` sem remover a primeira → erro `series limit reached`
- ✅ Remover a série anterior (`remover_serie`) e buscar outra

```python
# Free: remover antes de buscar nova série
remover_serie(uri="ct://series/binance/BTCUSDT/15m")
buscar_serie(provider="binance", symbol="ETHUSDT", timeframe="15m")
```

No Premium: até 100 séries simultâneas, com LRU eviction automática.

## Qual a diferença entre `snake_case` e `camelCase`?

| Contexto | Notação | Exemplo |
|---|---|---|
| Protocolo MCP (direto) | `snake_case` | `buscar_serie`, `ct_backtest` |
| SDK TypeScript | `camelCase` | `buscarSerie`, `ctBacktest` |
| Scripts Python | `snake_case` | `buscar_serie` |

A IA e o protocolo MCP usam `snake_case`. O SDK TypeScript (app desktop) converte para `camelCase`. Em scripts Python, use `snake_case`.

## Como funciona o sistema de URIs `ct://`?

Tudo no CT Lab é referenciado por uma URI `ct://`:

| URI | O que é |
|---|---|
| `ct://series/binance/BTCUSDT/15m` | Série OHLCV raw |
| `ct://derived/btc_rsi` | Série derivada (indicador) |
| `ct://backtest/a1b2c3` | Resultado de backtest |
| `ct://models/meu_modelo` | Modelo ML treinado |
| `ct://license/info` | Status da licença |

> Ferramentas MCP **nunca** retornam linhas de dados crusas — retornam uma URI + metadados. Para ler os dados, use a URI como resource (`ct://series/.../tail/100`).

## Preciso instalar algo além do CT Lab Desktop?

Não. O `ct-mcp-server` acompanha o app desktop — é iniciado automaticamente.

Para **ML** (Premium), você precisa do `uv` instalado:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Para **Python custom** (indicadores custom em Python), o `uv` também é necessário. Rhai é embutido — zero dependências.

## Como passo parâmetros para indicadores?

Cada indicador expõe seus parâmetros na chamada da tool MCP. Os defaults são aplicados se você omitir:

```python
# RSI com período padrão (14)
rsi(uri="ct://series/binance/BTCUSDT/15m")
# → ct://derived/rsi_...

# RSI com período customizado
rsi(uri="ct://series/binance/BTCUSDT/15m", period=21)
# → ct://derived/rsi_...

# MACD com fast/slow/signal customizados
macd(uri="ct://series/binance/BTCUSDT/15m", fast=10, slow=30, signal=5)
```

| Indicador | Parâmetros | Defaults |
|---|---|---|
| `rsi`, `sma`, `ema`, `bollinger` | `period` | 14, 20, 20, 20 |
| `macd` | `fast`, `slow`, `signal` | 12, 26, 9 |
| `stochastic` | `period`, `smooth` | 14, 3 |
| `ichimoku` | `tenkan`, `kijun`, `senkou` | 9, 26, 52 |
| `psar` | `af_step`, `af_max` | 0.02, 0.2 |
| `adx` | `period` | 14 |

> Consulte o catálogo vivo para a lista completa: `ct://indicators/catalog`

## Qual a diferença entre `materializar_indicador` e `montar_pipeline_indicadores`?

| Ferramenta | Quando usar | Complexidade |
|---|---|---|
| `materializar_indicador` | Uma expressão Rhai única (ex: `sma(rsi(close, 14), 5)`) | Simples |
| `montar_pipeline_indicadores` | Múltiplos steps com DAG, ops declarativas, composição | Avançada |

```python
# Simples: materializar_indicador
materializar_indicador(
    fonte="ct://series/binance/BTCUSDT/15m",
    name="btc_rsi_sma",
    receita="sma(rsi(close, 14), 5)"
)

# Avançado: pipeline com múltiplos steps
montar_pipeline_indicadores({
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "sinal_cross",
    "output": "$sinal",
    "steps": [
        { "id": "fast", "op": "sma", "source": "$anchor", "period": 9 },
        { "id": "slow", "op": "sma", "source": "$anchor", "period": 21 },
        { "id": "cruz", "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" }
    ]
})
```

## `close[0]` significa "barra atual" ou "primeiro elemento"?

Depende do contexto:

| Contexto | `close[0]` | `close` |
|---|---|---|
| **Backtest** (estratégia Rhai) | Valor escalar da barra atual | Não usado diretamente — sempre indexado |
| **materializar_indicador** (Rhai vetorizado) | Primeiro elemento da série | Série completa (array) |

```rhai
// Backtest: close[0] = barra atual, close[1] = anterior
if close[0] > close[1] { comprado(1.0) }

// Vetorizado: close = série completa, sma opera sobre todo o array
sma(close, 14)
```

## Como uso `comparar` vs `condicional` na pipeline?

| Op | O que faz | Exemplo |
|---|---|---|
| `comparar` | Série × série → 0/1 | SMA curta cruzou SMA longa? |
| `condicional` | Série vs escalar → 0/1 | RSI < 30? |

```json
// comparar: série × série
{ "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" }

// condicional: série vs escalar (RSI < 30)
{ "op": "condicional", "condicao": "$rsi", "coluna_condicao": "rsi", "entao": {"escalar": 1.0}, "senao": {"escalar": 0.0} }
```

> **Atenção:** `comparar` é sempre série×série. Para threshold contra escalar (`rsi > 30`), use `condicional`.

## Como criar um indicador customizado?

Use o step `custom` na pipeline. Rhai para lógica simples (sem dependências), Python para cálculos complexos:

```json
// Rhai custom (mais rápido, sem dependências)
{
  "id": "meu_sinal",
  "op": "custom",
  "script": "let r = rsi(close, par[\"p\"]); if r > 70.0 { -1.0 } else if r < 30.0 { 1.0 } else { 0.0 }",
  "entradas": [{ "alias": "close", "fonte": "$anchor", "coluna": "close" }],
  "parametros": { "p": 14 },
  "coluna_saida": "sinal"
}

// Python custom (com numpy, scipy, etc.)
{
  "id": "zscore_py",
  "op": "custom",
  "script": "import numpy as np\ndef calcular(close, par):\n    r = np.array(close)\n    return {\"z\": ((r - r.mean()) / r.std()).tolist()}",
  "entradas": [{ "alias": "close", "fonte": "$anchor", "coluna": "close" }],
  "deps": ["numpy"]
}
```

## Como funcionam os parâmetros em scripts Rhai de backtest?

Variáveis declaradas com `let` persistem entre barras. Parâmetros são acessíveis via `par["nome"]`:

```rhai
// Estado persiste entre barras
let entrada = 0.0;
let stop = 0.0;

if posicao == 0.0 && ind["rsi"][0] < par["limite"] {
    entrada = close[0];
    stop = close[0] * (1.0 - par["stop_pct"]);
    comprado(1.0)
} else if posicao > 0.0 && close[0] < stop {
    zerado()
} else {
    comprado(1.0)
}
```

| Variável | O que é |
|---|---|
| `close[0]`, `high[0]`, `low[0]` | Barra atual |
| `close[1]` | Barra anterior |
| `posicao` | Posição atual (0=zerado, >0=comprado) |
| `ind["alias"][0]` | Valor atual do indicador |
| `par["nome"]` | Parâmetro da estratégia |

> Sempre use `f64` (1.0) não `int` (1) nos argumentos de Rhai.

> Voltar para: [README](./README.pt.md)
