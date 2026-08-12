# 01 — Cruzamento de SMA: A Primeira Estratégia

> **Nível:** Iniciante · **Tempo:** 15 min · **Pré-requisitos:** [Instalação](../docs/01-instalacao/) completa, MCP conectado

O cruzamento de médias móveis é a estratégia sistemática mais antiga e simples que existe — perfeito como primeiro exemplo. Você vai buscar dados reais, calcular indicadores, montar uma pipeline declarativa, rodar múltiplos backtests e descobrir por que "simples" não significa "lucrativo".

---

## O que é cruzamento de SMA?

A **Simple Moving Average (SMA)** é a média aritmética dos preços de fechamento dos últimos *N* candles. Suaviza o ruído de preço para revelar a tendência subjacente.

Quando você plota duas SMAs de períodos diferentes, seus **cruzamentos** funcionam como sinais:

| Sinal | Condição | Ação |
|-------|-----------|------|
| **Cruzamento dourado** | SMA rápida cruza *acima* da SMA lenta | **Comprar** (altista) |
| **Cruzamento de morte** | SMA rápida cruza *abaixo* da SMA lenta | **Vender** (baixista) |

```
Preço  ─────╮         ╭─────────
            ╰───╮  ╭──╯
SMA(9)  ────╰──╯────────────
SMA(21) ────────────────────

              ↑ Cruzamento dourado (comprar)
```

---

## Passo 1 — Buscar a série BTCUSDT 15m

**Chat (IA):**
> Busque BTCUSDT em 15m da Binance

**Tool call (MCP):**
```json
{
  "name": "buscar_binance",
  "arguments": { "symbol": "BTCUSDT", "interval": "15m" }
}
```

**Retorno real:**
```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": 1785611700,
  "last_ts": 1786510800,
  "evicted_series": 0
}
```

A série fica disponível em `ct://series/binance/BTCUSDT/15m`. O cache já continha candles de execução anteriores — verificando os metadados:

```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "kind": "raw",
  "columns": ["open", "high", "low", "close", "volume"],
  "row_count": 1724,
  "first_ts": 1784960100,
  "last_ts": 1786510800
}
```

**1.724 candles** de BTCUSDT em 15m — aproximadamente 18 dias de mercado contínuo.

---

## Passo 2 — Calcular SMAs individuais

Antes de montar a pipeline, vamos olhar os valores das SMAs isolamente para desenvolver intuição.

**Chat (IA):**
> Calcule SMA de 9 e SMA de 21 sobre o BTCUSDT 15m

**Tool calls:**
```json
[
  { "name": "sma", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 9, "name": "btc_sma9" } },
  { "name": "sma", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 21, "name": "btc_sma21" } }
]
```

**Retorno (SMA 9):**
```json
{
  "uri": "ct://derived/btc_sma9",
  "row_count": 1724,
  "value_names": ["sma"],
  "latest": [63774.85]
}
```

**Retorno (SMA 21):**
```json
{
  "uri": "ct://derived/btc_sma21",
  "row_count": 1724,
  "value_names": ["sma"],
  "latest": [63769.40]
}
```

No último candle, SMA(9) = **63.774,85** e SMA(21) = **63.769,40**. A rápida está ligeiramente acima da lenta — estado altista marginal. Os dois estão muito próximos, típico antes de um cruzamento.

> **Nota:** Cada SMA calculada gera uma série derivada persistida em `ct://derived/<nome>`. Você pode reutilizar essas séries depois sem recalcular.

---

## Passo 3 — Montar pipeline declarativa com cruzamento

Em vez de dois cálculos separados, podemos usar a pipeline declarativa para produzir uma série derivada com o sinal de cruzamento completo.

**Chat (IA):**
> Monte uma pipeline que calcule SMA 9, SMA 21 e um sinal de cruzamento

**Tool call:**
```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_cruzamento_sma",
    "output": "$sinal",
    "steps": [
      { "id": "sma9", "op": "sma", "source": "$anchor", "period": 9 },
      { "id": "sma21", "op": "sma", "source": "$anchor", "period": 21 },
      {
        "id": "cruz_acima",
        "op": "comparar",
        "esquerda": "$sma9",
        "coluna_esquerda": "sma",
        "direita": "$sma21",
        "coluna_direita": "sma",
        "operador": "cruza_acima",
        "coluna_saida": "cruz_acima"
      },
      {
        "id": "cruz_abaixo",
        "op": "comparar",
        "esquerda": "$sma9",
        "coluna_esquerda": "sma",
        "direita": "$sma21",
        "coluna_direita": "sma",
        "operador": "cruza_abaixo",
        "coluna_saida": "cruz_abaixo"
      },
      {
        "id": "sinal",
        "op": "condicional",
        "condicao": "$cruz_acima",
        "coluna_condicao": "cruz_acima",
        "entao": { "escalar": 1.0 },
        "senao": { "escalar": -1.0 },
        "coluna_saida": "sinal"
      }
    ]
  }
}
```

**Retorno real:**
```json
{
  "uri": "ct://derived/btc_cruzamento_sma",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1724,
  "first_ts": 1784960100,
  "last_ts": 1786510800,
  "steps_executed": 5
}
```

> **Detalhe importante:** quando você compara duas séries derivadas (não a coluna `close` do anchor), **deve** especificar `coluna_esquerda` e `coluna_direita` com o nome da coluna produzida pelo step anterior. No caso, as SMAs produzem a coluna `sma`, então usamos `coluna_esquerda: "sma"` e `coluna_direita: "sma"`. Se omitido, o motor tenta usar `close` — que não existe numa série derivada de SMA.

A pipeline executou **5 steps** e foi persistida em `ct://derived/btc_cruzamento_sma`. Essa série contém o sinal: `1` quando a rápida cruzou acima da lenta, `-1` quando cruzou abaixo.

---

## Passo 4 — Backtest SMA 9/21 (sem taxas)

O momento da verdade: vamos rodar o backtest.

**Chat (IA):**
> Rode um backtest: compre quando SMA 9 > SMA 21, senão zerado. Capital 1000, sem fee.

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma9\"][0] > ind[\"sma21\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma9": { "receita": "sma(close, 9)" },
      "sma21": { "receita": "sma(close, 21)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "sma_cross_9_21_nofee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "uri": "ct://backtest/sma_cross_9_21_nofee",
  "num_trades": 46,
  "pnl_total": -44.82,
  "pnl_bruto": 45.26,
  "fees_totais": 0,
  "retorno_total": -0.0448,
  "sharpe": 0.043,
  "sortino": 0.326,
  "calmar": -0.453,
  "win_rate": 0.3696,
  "profit_factor": 1.010,
  "avg_win": 275.45,
  "avg_loss": -159.91,
  "payoff_ratio": 1.72,
  "drawdown_max": 1.34,
  "exposicao": 0.5238,
  "num_wins": 17,
  "num_losses": 29,
  "max_wins_seguidos": 3,
  "max_perdas_seguidas": 7
}
```

Sem taxas, a estratégia é essencialmente break-even:

- **PnL bruto:** +$45,26 (um edge minúsculo)
- **Win rate:** 36,96% (apenas ~37% dos trades ganham)
- **Profit factor:** 1,010 (mal acima de 1,0)
- **Payoff ratio:** 1,72 (ganhos são 1,72× maiores que perdas)

A estratégia é marginalmente lucrativa antes de custos. Mas "marginalmente lucrativa antes de custos" é um lugar perigoso — porque os custos estão prestes a mudar tudo.

---

## Passo 5 — O impacto das taxas

Trading real tem taxas. A Binance cobra aproximadamente **0,1% por trade** (taker fee). Vamos re-rodar o mesmo backtest com `fee_pct: 0.001`.

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma9\"][0] > ind[\"sma21\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma9": { "receita": "sma(close, 9)" },
      "sma21": { "receita": "sma(close, 21)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "sma_cross_9_21_fee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "uri": "ct://backtest/sma_cross_9_21_fee",
  "num_trades": 46,
  "pnl_total": -6080.91,
  "pnl_bruto": 45.26,
  "fees_totais": 5908.45,
  "sharpe": 0.025,
  "sortino": 0.039,
  "win_rate": 0.2174,
  "profit_factor": 0.353,
  "avg_win": 320.55,
  "avg_loss": -251.91,
  "payoff_ratio": 1.27,
  "drawdown_max": 5.89
}
```

### A diferença brutal

| Métrica | Sem fee | Com fee 0,1% |
|---|---------|--------------|
| **PnL Total** | -$44,82 | **-$6.080,91** |
| **PnL Bruto** | $45,26 | $45,26 |
| **Fees** | $0,00 | **$5.908,45** |
| **Win Rate** | 36,96% | 21,74% |
| **Profit Factor** | 1,010 | 0,353 |
| **Drawdown Máx** | 1,34% | 5,89% |

O edge bruto de **$45,26** é completamente destruído por **$5.908,45** em taxas.

### Por que tanto em fees?

Cada trade custa **0,1% do valor nocional**. Com BTC a ~$64.000, cada trade custa aproximadamente:

```
$64.000 × 0,001 = $64 na entrada + $64 na saída = ~$128 round-trip
```

Com **46 trades** (cada trade = entrada + saída):

```
$128 × 46 = $5.888 ≈ $5.908,45 ✓
```

O edge bruto de $45 representa **0,76%** do custo em taxas. As taxas são **131× maiores** que o edge!

---

## Passo 6 — Variar os parâmetros

Será que períodos diferentes produzem resultados melhores? Vamos testar duas variantes extremas e comparar com Buy & Hold. Todas com fee 0,1%.

### Variante A: SMA 5/20 (mais rápido, mais trades)

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma5\"][0] > ind[\"sma20\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma5": { "receita": "sma(close, 5)" },
      "sma20": { "receita": "sma(close, 20)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "sma_cross_5_20_fee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "num_trades": 60,
  "pnl_total": -11522.77,
  "pnl_bruto": -3633.30,
  "fees_totais": 7708.06,
  "sharpe": -0.014,
  "win_rate": 0.1833,
  "profit_factor": 0.243,
  "drawdown_max": 10.81
}
```

### Variante B: Golden Cross 50/200 (mais lento, menos trades)

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma50\"][0] > ind[\"sma200\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma50": { "receita": "sma(close, 50)" },
      "sma200": { "receita": "sma(close, 200)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "golden_cross_50_200_fee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "num_trades": 7,
  "pnl_total": -3916.78,
  "pnl_bruto": -3018.49,
  "fees_totais": 898.29,
  "sharpe": -0.026,
  "win_rate": 0.1429,
  "profit_factor": 0.052,
  "drawdown_max": 2.31,
  "exposicao": 0.6363
}
```

### Variante C: Buy & Hold (benchmark)

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "comprado(1.0)",
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "buy_hold"
  }
}
```

**Retorno real (resumo):**
```json
{
  "num_trades": 0,
  "pnl_total": -281.50,
  "retorno_total": -0.2815,
  "sharpe": -0.024,
  "drawdown_max": 1.25,
  "exposicao": 0.9994
}
```

> O Buy & Hold mostra que o mercado caiu no período (retorno = **-28,15%**). Isso é contexto importante: a estratégia lutava contra um mercado em baixa.

---

## Tabela comparativa — 5 estratégias

| # | Estratégia | Trades | PnL Total | PnL Bruto | Fees | Sharpe | Win Rate | PF | Payoff | DD Max | Exposição |
|---|-----------|--------|-----------|-----------|------|--------|----------|----|--------|--------|-----------|
| 1 | SMA 9/21 (sem fee) | 46 | -$44,82 | $45,26 | $0 | 0,043 | 36,96% | 1,010 | 1,72 | 1,34% | 52,38% |
| 2 | SMA 9/21 (fee 0,1%) | 46 | -$6.080,91 | $45,26 | $5.908 | 0,025 | 21,74% | 0,353 | 1,27 | 5,89% | 52,38% |
| 3 | SMA 5/20 (fee 0,1%) | 60 | -$11.522,77 | -$3.633 | $7.708 | -0,014 | 18,33% | 0,243 | 1,08 | 10,81% | 52,32% |
| 4 | Golden Cross 50/200 | 7 | -$3.916,78 | -$3.018 | $898 | -0,026 | 14,29% | 0,052 | 0,31 | 2,31% | 63,63% |
| 5 | Buy & Hold | 0 | -$281,50 | — | — | -0,024 | — | — | — | 1,25% | 99,94% |

### Observações da tabela

1. **Fees dominam** as estratégias de SMA curto. SMA 9/21 produz $45 de edge bruto mas paga $5.908 em taxas — a razão fee/edge é **131:1**.

2. **Cruzamentos rápidos pioram tudo.** SMA 5/20 gera 60 trades (vs 46 do 9/21), paga $7.708 em taxas, *e* tem PnL bruto negativo (-$3.633). Mais trading = mais taxas + mais whipsaw.

3. **Golden Cross não ajuda.** Transa pouco (7 trades, $898 em taxas), mas o PnL bruto já é deep negativo (-$3.018). Os sinais são lentos demais para 15m — quando a SMA 200 cruza, o movimento já acabou. E 7 trades é estatisticamente insignificante.

4. **Buy & Hold foi o menos desastroso.** Perdeu $281 no período, mas superou todas as variantes SMA por larga margem. O mercado estava em baixa, mas apenas segurar foi muito mais barato que churnar capital em cruzamentos.

5. **Exposição importa.** SMA 9/21 esteve no mercado só 52% do tempo, mas perdeu muito mais que Buy & Hold a 99,94% de exposição. Ser "seletivo" não ajudou — só adicionou custo de transação.

---

## Interpretação — O que cada métrica significa

### PnL Total vs PnL Bruto
- **PnL Bruto** = soma de lucros e perdas por trade *antes* de taxas.
- **PnL Total** = PnL Bruto − Fees.
- O gap entre os dois é o **custo real de trading**. Para SMA 9/21 com fee, o gap é $5.953 — 98% da perda total vem de taxas, não de mau sinal.

### Sharpe Ratio
Mede retorno ajustado ao risco. Sharpe de 0,043 (sem fee) é quase zero — indistinguível de aleatoriedade. Com fee, cai para 0,025. Para SMA 5/20 e Golden Cross, fica negativo.

### Sortino Ratio
Como Sharpe, mas só penaliza volatilidade *negativa*. Sortino de 0,326 no caso sem fee parece melhor, mas ainda indica que a estratégia não gera retornos ajustados ao risco significativos.

### Win Rate
A % de trades lucrativos. 36,96% significa que ~63% dos trades perdem dinheiro. Mas win rate isolado não diz muito — o que importa é a **combinação** de win rate e payoff ratio.

### Profit Factor (PF)
Ganhos totais ÷ perdas totais.
- PF > 1,0 = lucrativo (antes de taxas)
- PF > 1,5 = sólido
- PF > 2,0 = excelente
- PF < 1,0 = deficitário

SMA 9/21 tem PF = 1,010 *sem taxas* — essencialmente cara ou coroa com leve edge. Com taxas, PF desaba para 0,353.

### Payoff Ratio
Ganho médio ÷ perda média. $275,45 / $159,91 = 1,72. Ganhos são 1,72× maiores que perdas — a única qualidade redentora da estratégia. Mas não basta para superar win rate baixo + taxas.

### Drawdown Máximo
A maior queda do equity de pico a vale. O drawdown de 5,89% com fee é controlável, mas é prejuízo *em cima* de perdas.

### Exposição
A proporção do tempo em que a estratégia está posicionada. SMA 9/21 está investida ~52% do tempo. Exposição baixa pode ser bom se o sinal é seletivo — mas aqui só significa que pagamos taxas para acertar metade do tempo.

---

## Por que SMA crossover não funciona bem?

Esta é a lição pedagógica mais importante deste exemplo.

### 1. SMA é um indicador atrasado

A SMA é — por definição — uma média de preços *passados*. Quando a rápida cruza a lenta, a mudança de tendência **já aconteceu**. Você está reagindo, não prevendo. Em mercados choppy (que 15m de crypto costuma ser), esse atraso significa comprar perto de topos locais e vender perto de fundos locais.

### 2. Whipsaw em mercado lateral

Cruzamentos de SMA funcionam lindamente em **mercados de tendência** — você pega um sinal limpo e surfa a tendência. Mas em **mercados laterais** (que dominam timeframes curtos), as duas SMAs oscilam em torno uma da outra, gerando falso após falso. Cada falso é um trade round-trip, e cada trade custa ~$128 em taxas.

### 3. A matemática de fees é brutal

```
Edge bruto por trade = $45,26 / 46 trades ≈ $0,98 por trade
Fee por round-trip    = ~$128 por trade

Razão edge/fee        = $0,98 / $128 = 0,0077

Você precisaria de um edge 130× maior só pra empatar após taxas.
```

Nenhuma estratégia de SMA crossover em 15m vai produzir $128 de edge bruto por trade. O edge por trade é tipicamente de um dígito — ordens de grandeza abaixo do threshold de fee.

### 4. O ativo não tende o suficiente em 15m

Num timeframe de 15m, BTC não tende de forma suave por longos períodos dentro de 2-3 semanas. Os dados mostram um mercado em declínio (-28,15% no Buy & Hold), e SMA crossover é trend-following — desenhado para capturar movimentos direcionais sustentados. Quando esses movimentos são choppy e curtos, a estratégia sangra.

### 5. SMA pondera todos os preços igualmente

Uma SMA de 21 períodos dá o mesmo peso ao candle de 21 barras atrás e ao candle mais recente. Na prática, price action recente é muito mais relevante. É por isso que indicadores como **EMA** (Exponential Moving Average) existem — mas mesmo EMA não resolve o problema fundamental das taxas.

---

## Variações pra experimentar

| # | Variação | Como | Hipótese |
|---|----------|-----|----------|
| 1 | **EMA no lugar de SMA** | Trocar `sma(close, N)` por `ema(close, N)` nas receitas | EMA reage mais rápido; pode capturar tendências mais cedo |
| 2 | **Timeframe maior (1h ou 4h)** | Re-buscar com `interval: "1h"` | Menos candles, menos trades, edge maior por trade — fees viram proporcionais menores |
| 3 | **Filtro ADX** | Adicionar ADX no `indicadores_receitas`; só operar quando ADX > 25 | Filtra whipsaws em mercado sem tendência |
| 4 | **Filtro ATR crescente** | Só entrar quando ATR está subindo (expansão de vol) | Sinais de cruzamento em baixa vol são frequentemente falsos |
| 5 | **Fee menor** | Reduzir `fee_pct` para 0,0005 (maker) ou 0,0001 (VIP) | Mostra como a estrutura de taxas afeta viabilidade — em que fee SMA 9/21 empata? |
| 6 | **Apenas longo** | Estratégia só compra, nunca vende | Evita shortar em downtrends; pode reduzir whipsaw |
| 7 | **Outros ativos** | Tentar `ETHUSDT` ou `SOLUSDT` | Ativos diferentes tendem de forma diferente |

> **Desafio experimental:** Qual a taxa mínima em que SMA 9/21 fica lucrativo? Itere `fee_pct` de 0,0001 a 0,001 em passos de 0,0001 e observe o PnL. Você provavelmente vai descobrir que o breakeven está em ~0,0001–0,0003 — dentro do territory de maker fee com desconto VIP.

---

## Próximos passos

Você construiu, testou e analisou sua primeira estratégia sistemática. Para onde ir agora:

1. **[Receita 02 — RSI com filtro ADX](./02-rsi-filtro-adx.pt.md)** — Adiciona um filtro de tendência para reduzir falsos sinais do RSI.

2. **[Receita 03 — Backtest com lib grupo](./03-lib-grupo-backtest.pt.md)** — Execução sofisticada com entradas escalonadas, stops OCO e trailing.

3. **[Exemplo 04 — Teste de sobrevivência](./04-teste-sobrevivencia.pt.md)** — Como saber se sua estratégia sobrevive fora da amostra.

4. **[Exemplo 08 — Microestrutura: TFI](./08-tfi-backtest.pt.md)** — Indo além de OHLCV: indicadores de fluxo de ordens.

5. **[Exemplo 12 — Rhai inline](./12-rhai-inline-backtest.pt.md)** — 8 estratégias clássicas escritas em Rhai inline, comparadas lado a lado.

---

> **Lição final:** SMA crossover é uma estratégia que *parece* que deveria funcionar — é intuitiva, visual, tem um século de literatura. Mas o abismo entre "parece razoável" e "é lucrativo após custos" é enorme. Este exemplo é sua primeira exposição a esse abismo. Cada exemplo subsequente explora técnicas para estreitá-lo.
>
> **A lição não é "SMA crossover é inútil".** A lição é: **sempre inclua taxas no backtest, entenda sua razão edge/fee, e reconheça que em timeframes curtos com edge moderado, custo de transação é o determinante dominante de lucratividade.**

---

*Todos os resultados são saídas reais de uma sessão MCP do CT Lab em dados BTCUSDT 15m cobrindo 1.724 candles. Nenhum valor é ilustrativo.*
