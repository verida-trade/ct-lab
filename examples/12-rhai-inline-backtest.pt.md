# 12 — Rhai Inline: Estratégias sem Arquivo

> **Nível:** Intermediário · **Pré-requisitos:** [Fork da Doutrina](./11-fork-doutrina.pt.md), [Estratégia Rhai](../docs/05-backtest/03-estrategia-rhai.pt.md)

Em todos os exemplos anteriores, você escreveu estratégias como `estrategia_script` — uma string Rhai passada direto no JSON do `ct_backtest`. Mas nunca parou para entender o contrato: o que está disponível dentro desse script? Quais variáveis você pode ler? Quais funções pode chamar?

Este exemplo é o manual prático do **Rhai inline** — a linguagem de scripting que executa dentro do backtest. Você vai ver 8 estratégias clássicas escritas inline, comparar seus resultados, e entender a anatomia do contrato Rhai do CT Lab.

> **Rhai** é uma linguagem de scripting embutida em Rust, sandboxed, sem I/O, type-safe. No CT Lab, o backtest executa o script barra por barra, injetando preço, posição e indicadores. O retorno do script vira a decisão de execução.

---

## O contrato Rhai

Cada barra, o backtest chama seu script com as seguintes variáveis disponíveis:

### Variáveis de mercado (séries indexadas)

| Variável | Tipo | Descrição |
|---|---|---|
| `open[0]` | float | Preço de abertura do candle atual |
| `high[0]` | float | Preço máximo |
| `low[0]` | float | Preço mínimo |
| `close[0]` | float | Preço de fechamento |
| `volume[0]` | float | Volume |
| `open[1]` | float | candles anteriores (`[1]`, `[2]`, ...) |

### Variáveis de estado

| Variável | Tipo | Descrição |
|---|---|---|
| `posicao` | float | Posição atual (lotes líquidos; > 0 comprado, < 0 vendido) |
| `estado` | map | Estado persistente entre barras (objeto mutável) |
| `ordens` | map | Ordens do candle atual (preenchidas) |
| `par` | map | Parâmetros da estratégia (`par["nome"]`) |

### Variáveis de indicadores

| Variável | Tipo | Descrição |
|---|---|---|
| `ind["alias"][0]` | float | Valor do indicador no candle atual |
| `ind["alias"][1]` | float | Valor no candle anterior |

### Funções de decisão (retorno obrigatório)

| Função | Descrição |
|---|---|
| `comprado(qtd)` | Comprar/manter `qtd` lotes |
| `vendido(qtd)` | Vender/manter `qtd` lotes |
| `zerado()` | Zerar posição |
| `decisao(alvo, ordens)` | Decisão com ordens OCO (para gestor) |

### Funções de ordem (para gestor `grupo`)

| Função | Descrição |
|---|---|
| `limite_compra(id, lote, preco)` | Ordem limit de compra |
| `limite_venda(id, lote, preco)` | Ordem limit de venda |
| `stop_compra(id, lote, preco)` | Stop de compra |
| `stop_venda(id, lote, preco)` | Stop de venda |
| `oco(grp, ordem)` | Ordem OCO (one-cancels-other) |

### Indicadores inline (receitas)

Indicadores podem ser declarados como **receita** — fórmula Rhai vetorizada computada sobre a série de preço, sem pré-materializar:

```json
{
  "indicadores_receitas": {
    "rsi": { "receita": "rsi(close, 14)" },
    "sma_rapida": { "receita": "sma(close, 20)", "parametros": { "period": 20 } }
  }
}
```

> O motor materializa cada receita sobre a série e injeta como `ind["alias"]`. A receita tem acesso às colunas (`close`, `open`, `high`, `low`, `volume`) e a `par` (parâmetros da receita). A estratégia tem acesso a `ind["alias"]` e a `par` (parâmetros da estratégia).

---

## Estratégia 1 — SMA Crossover (20/50)

O cruzamento de médias móveis: quando a média rápida cruza acima da lenta, compra; quando cruza abaixo, vende.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "sma_rapida": { "receita": "sma(close, 20)" },
      "sma_lenta": { "receita": "sma(close, 50)" }
    },
    "estrategia_script": "if ind[\"sma_rapida\"][0] > ind[\"sma_lenta\"][0] { comprado(1.0) } else { vendido(1.0) }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_smacross_fee"
  }
}
```

### Anatomia

```
开盘: sma_rapida = sma(close, 20)
      sma_lenta  = sma(close, 50)
决策: if sma_rapida > sma_lenta → comprado(1.0)
      else → vendido(1.0)
```

Sempre posicionado — nunca neutro. Se a média rápida está acima da lenta, está comprado; senão, vendido. Trend-following binário.

---

## Estratégia 2 — RSI Extreme (14)

RSI < 30 = sobreventa, compra. RSI > 70 = sobrecompra, vende. Otherwise, neutro.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_rsi_fee"
  }
}
```

Mean reversion: compra quando o mercado caiu demais, vende quando subiu demais. Só opera em extremos — o resto do tempo fica de fora.

---

## Estratégia 3 — Bollinger Breakout

Preço rompe a banda superior = compra (breakout). Preço rompe a banda inferior = venda.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r12_bollinger",
    "estrategia_script": "if close[0] > ind[\"upper\"][0] { comprado(1.0) } else if close[0] < ind[\"lower\"][0] { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_boll_fee"
  }
}
```

> Aqui usamos `indicadores` (série derivada pré-materializada via pipeline) em vez de `indicadores_receitas`, porque o Bollinger produz múltiplas colunas (`upper`, `lower`, `middle`) que não podem ser expressas como uma única receita.

---

## Estratégia 4 — MACD Histogram

MACD acima do sinal = momentum positivo, compra. Abaixo = momentum negativo, venda.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r12_macd",
    "estrategia_script": "if ind[\"macd\"][0] > ind[\"signal\"][0] { comprado(1.0) } else { vendido(1.0) }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_macd_fee"
  }
}
```

Sempre posicionado, igual ao SMA crossover — mas usando a direção do histograma MACD como gatilho.

---

## Estratégia 5 — RSI + SMA200 (combo)

Trend filter + mean reversion: só compra se preço está acima da SMA200 (tendência de alta) e RSI < 40 (pullback). Vice-versa para venda.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" },
      "sma200": { "receita": "sma(close, 200)" }
    },
    "estrategia_script": "if close[0] > ind[\"sma200\"][0] && ind[\"rsi\"][0] < 40.0 { comprado(1.0) } else if close[0] < ind[\"sma200\"][0] && ind[\"rsi\"][0] > 60.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_combo_fee"
  }
}
```

Combina dois conceitos: tendência (SMA200) e momentum (RSI). Ofiltro de tendência reduz trades — só opera pullbacks na direção da tendência.

---

## Estratégia 6 — ATR Breakout

Preço rompe SMA20 ± 1 ATR = breakout de volatilidade. Compra acima, vende abaixo.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "atr": { "receita": "atr(high, low, close, 14)" },
      "sma_close": { "receita": "sma(close, 20)" }
    },
    "estrategia_script": "if close[0] > ind[\"sma_close\"][0] + ind[\"atr\"][0] { comprado(1.0) } else if close[0] < ind[\"sma_close\"][0] - ind[\"atr\"][0] { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_atr_fee"
  }
}
```

A banda de ATR se adapta à volatilidade: em mercados calmos, a banda é estreita (poucos sinais); em mercados voláteis, é larga (sinais só em movimentos grandes).

---

## Estratégia 7 — Price Action (sem indicadores)

Higher high + candle de alta = compra. Lower low + candle de baixa = venda. Sem nenhum indicador.

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if high[0] > high[1] && close[0] > open[0] { comprado(1.0) } else if low[0] < low[1] && close[0] < open[0] { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_price_fee"
  }
}
```

A estratégia mais pura: usa apenas `open`, `high`, `low`, `close` — sem nenhum indicador. Mostra que o contrato Rhai dá acesso direto ao OHLCV sem processamento.

---

## Estratégia 8 — RSI Parametrizado

Igual à Estratégia 2, mas com thresholds parametrizados via `parametros`:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, par[\"period\"])", "parametros": { "period": 14 } }
    },
    "estrategia_script": "if ind[\"rsi\"][0] < par[\"oversold\"] { comprado(1.0) } else if ind[\"rsi\"][0] > par[\"overbought\"] { vendido(1.0) } else { zerado() }",
    "parametros": { "oversold": 25, "overbought": 75 },
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r12_rsi_param_fee"
  }
}
```

> Dois níveis de parâmetros: `parametros` na receita (`period`) e `parametros` na estratégia (`oversold`, `overbought`). A receita tem seu próprio `par`, e a estratégia tem o seu. Ambos são injetados no `par` do respectivo sandbox.

---

## Resultados

### Com taxas (0.1%)

| Estratégia | Trades | PnL | Win% | PF | Sharpe | Max DD | Exposição |
|---|---|---|---|---|---|---|---|
| SMA cross (20/50) | 77 | −$14.774 | 13.3% | 0.43 | 0.021 | 147.8% | 99.9% |
| RSI extreme (30/70) | 107 | −$13.511 | 9.3% | 0.22 | 0.007 | 135.1% | 14.3% |
| Bollinger breakout | 148 | −$18.478 | 4.7% | 0.11 | 0.056 | 184.8% | 10.5% |
| MACD histogram | 101 | −$18.926 | 13.9% | 0.25 | 0.028 | 190.3% | 99.9% |
| RSI + SMA200 combo | 76 | −$9.732 | 11.8% | 0.29 | 0.029 | 97.3% | 9.7% |
| ATR breakout | 44 | −$5.597 | 18.2% | 0.43 | 0.043 | 56.0% | 4.4% |
| Price action | 822 | −$108.580 | 7.2% | 0.07 | 0.024 | 1085.8% | 82.4% |
| RSI parametrizado (25/75) | 78 | −$9.893 | 11.5% | 0.06 | −0.063 | 98.9% | 11.4% |
| Buy & Hold | 0 | −$296 | — | — | 0.003 | 28.9% | 99.9% |

### Sem taxas

| Estratégia | Trades | PnL | Win% | PF | Sharpe | Calmar | Max DD |
|---|---|---|---|---|---|---|---|
| SMA cross (20/50) | 77 | −$1.776 | 36.4% | 0.83 | 0.017 | 0.118 | 17.8% |
| RSI extreme (30/70) | 107 | +$1.369 | 77.6% | **1.82** | 0.013 | 48.49 | 12.4% |
| Bollinger breakout | 148 | +$1.627 | 72.3% | **1.55** | 0.013 | 238.77 | 12.4% |
| MACD histogram | 101 | −$5.926 | 22.8% | 0.60 | −0.026 | −1.45 | 69.1% |
| RSI + SMA200 combo | 76 | +$2.088 | 78.9% | **2.38** | 0.019 | 175.30 | 12.4% |
| ATR breakout | 44 | +$1.033 | 68.2% | **1.61** | 0.018 | 132.75 | 7.8% |
| Price action | 822 | +$2.932 | 35.9% | 1.02 | 0.009 | 4.30 | 43.3% |
| RSI parametrizado (25/75) | 78 | +$114 | 71.8% | **1.03** | 0.004 | 1.23 | 21.2% |
| Buy & Hold | 0 | −$168 | — | — | 0.003 | −1.03 | 28.6% |

---

## Análise

### Sem taxas: RSI + SMA200 é a melhor

O combo de tendência + mean reversion foi a estratégia mais lucrativa sem taxas: +$2.088, profit factor 2.38, win rate 78.9%, e Calmar de 175 — drawdown máximo de apenas 12.4%. O filtro de SMA200 corta entradas contra a tendência, mantendo só os pullbacks de qualidade.

### Sem taxas: Bollinger e RSI puro também têm edge

Bollinger breakout: PF 1.55, win 72.3%. RSI extreme: PF 1.82, win 77.6%. Ambos são estratégias de mean reversion que operam em extremos — relativamente poucos trades, alta taxa de acerto.

### Com taxas: todas perdem

Nenhuma estratégia supera 0.1% de taxa por lado. A energia de execução consome todo o edge. O ATR breakout é a menos pior (−$5.597) porque tem menos trades — 44 entradas vs 822 do price action.

### Price action é overtrading extremo

822 trades em 1712 candles (48% de turnover) com $111k em fees. Sem taxas tem PnL positivo (+$2.932) mas o turnover absurdo torna impossível operar com custo real.

> **A lição: simples funciona, mas o custo mata.** Estratégias clássicas (RSI, Bollinger, SMA cross) geram edge bruto positivo, mas nenhura supera 0.1% de taxa em BTC 15m. O caminho é reduzir turnover via filtros (regime do exemplo 09) ou usar um gestor que reduz trades (lib `grupo` do exemplo 11).

---

## Quando usar receita vs série derivada

| Abordagem | Quando usar | Vantagem |
|---|---|---|
| `indicadores_receitas` | Indicador retorna 1 coluna (RSI, SMA, ATR) | Sem pré-materializar — rápido |
| `indicadores` (série derivada) | Indicador retorna múltiplas colunas (Bollinger, MACD, ADX) | Colunas nomeadas | 
| Pipeline + `indicadores` | Múltiplos indicadores compostos | Um único URI, tudo alinhado |

> **Regra prática**: se o indicador tem uma coluna (RSI, ATR, SMA), use `indicadores_receitas`. Se tem múltiplas (Bollinger: upper/lower/middle, MACD: macd/signal, ADX: adx/plus_di/minus_di), materialize via pipeline e passe como `indicadores`.

---

## Próximos passos

- **Adicionar regime**: filtre as estratégias deste exemplo com ADX < 20 (lateralização) ou ADX > 25 (tendência) — veja [Regime + Modelo](./09-regime-modelo.pt.md).
- **Adicionar gestor**: combine as estratégias com a lib `grupo` para stops e trailing — veja [Fork da Doutrina](./11-fork-doutrina.pt.md).
- **Otimizar parâmetros**: use `parametros` para fazer grid search de períodos e thresholds — rode múltiplos backtests com diferentes valores e compare.
- **Salvar estratégia como arquivo**: se preferir, salve o script Rhai como arquivo `.rhai` e passe via `estrategia` em vez de `estrategia_script` — útil para scripts longos.

---

> Voltar para: [README](../README.md) · [Fork da Doutrina](./11-fork-doutrina.pt.md) · [Regime + Modelo](./09-regime-modelo.pt.md)

_Last updated: 2026-08-12_
