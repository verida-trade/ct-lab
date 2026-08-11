# Rhai Vetorizado

> `materializar_indicador` — expresse cálculos de indicadores em uma única linha de Rhai vetorizado.

O `ct-mcp-server` embute a linguagem [Rhai](https://rhai.rs/) como engine de scripts. A tool `materializar_indicador` permite escrever uma **expressão única** que encadeia indicadores sobre a série de entrada — sem precisar de pipeline multi-step.

---

## Tipo `Serie`

Os dados da série entram como variáveis no escopo Rhai:

| Variável | Tipo | O que é |
|---|---|---|
| `open` | `Serie` | Coluna open da raw |
| `high` | `Serie` | Coluna high |
| `low` | `Serie` | Coluna low |
| `close` | `Serie` | Coluna close |
| `volume` | `Serie` | Coluna volume |

### Operadores de `Serie`

```rhai
close + open          // série = série
close - open          // subtração
close * volume        // multiplicação
close / volume        // divisão
close[0]              // valor escalar da barra atual
close[1]              // valor escalar da barra anterior
avg(close)            // média de toda a série
sum(close)            // soma
highest(close, 20)    // maior valor dos últimos 20
lowest(close, 20)     // menor valor dos últimos 20
```

### Funções utilitárias

| Função | O que faz |
|---|---|
| `na()` | Not-a-Number |
| `nz(x)` | Null/NaN → 0 |
| `clamp(x, min, max)` | Limita valor |
| `sinal(x)` | Sinal: −1, 0, +1 |
| `abs(x)` | Valor absoluto |

### Indicadores como funções série→série

45+ indicadores estão disponíveis como funções que recebem e retornam `Serie`:

```rhai
sma(close, 14)                  // SMA de 14
ema(close, 9)                   // EMA de 9
rsi(close, 14)                 // RSI de 14
macd(close)["hist"]             // MACD → pega coluna hist (mapa)
bollinger(close)["upper"]       // Bollinger → pega upper
atr(high, low, close, 14)       // ATR de 14
obv(close, volume)             // OBV
stochastic(high, low, close)["k"]   // Stochastic %K
adx(high, low, close)["adx"]        // ADX
```

> Indicadores multi-saída (MACD, Bollinger, Stochastic, ADX, etc.) retornam um **mapa** — acesse com `["nome_da_coluna"]`.

---

## Exemplos

### SMA do RSI

```rhai
sma(rsi(close, 14), 5)
```

### Z-score do RSI

```rhai
let r = rsi(close, 14);
(r - avg(r)) / (r - avg(r))   // simplificado; use estatistica_rolling na pipeline pro desvio real
```

### Spread BTC-ETH (se ambas no input)

```rhai
btc_close / eth_close
```

> **Nota:** Para cross-asset, use `compor_serie` primeiro e depois `materializar_indicador` sobre a série synthetic.

### Sinal: RSI < 30 → +1, RSI > 70 → −1, senão 0

```rhai
let r = rsi(close, 14);
if r < 30.0 { 1.0 } else if r > 70.0 { -1.0 } else { 0.0 }
```

> **Atenção:** isto retorna um escalar (última barra). Para uma série completa, use `condicional` na pipeline ou retorne `#{ "sinal": r.map(|x| if x < 30.0 { 1.0 } else if x > 70.0 { -1.0 } else { 0.0 }) }`.

### Múltiplas colunas (mapa)

```rhai
#{
  "sma_curta": sma(close, 9),
  "sma_longa": sma(close, 21),
  "spread": sma(close, 9) - sma(close, 21)
}
```

---

## Chamada da tool

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_rsi_sma",
    "receita": "sma(rsi(close, 14), 5)"
  }
}
```

**Retorno:**
```json
{
  "uri": "ct://derived/btc_rsi_sma",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "valid_from_ts": 1784068200,
  "value_names": ["sma"]
}
```

### Com parâmetros

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_sinal_rsi",
    "receita": "if rsi(close, par[\"p\"]) < 30.0 { 1.0 } else if rsi(close, par[\"p\"]) > 70.0 { -1.0 } else { 0.0 }",
    "parametros": { "p": 14 }
  }
}
```

---

## Pipeline vs Rhai — quando usar qual?

| Situação | Use |
|---|---|
| 1 indicador | Tool direta (`sma`, `rsi`) |
| Expressão linear (1 linha) | **Rhai** (`materializar_indicador`) |
| 4+ steps com branches | **Pipeline** (`montar_pipeline_indicadores`) |
| Cross-asset | `compor_serie` + Rhai ou pipeline compose |

> **Regra:** Rhai é linear (uma expressão). Pipeline suporta árvores (múltiplos ramos convergindo).

---

> Próximo: [Cookbook de receitas](./07-cookbook.pt.md) · [Indicadores custom](./09-indicadores-custom.pt.md)
