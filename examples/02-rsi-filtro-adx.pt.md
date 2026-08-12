# 02 — RSI com Filtro ADX: Tendência e Momentum

> **Nível:** Intermediário · **Tempo:** 20 min · **Pré-requisitos:** [Exemplo 01 — Cruzamento de SMA](./01-cruzamento-sma.pt.md)

O RSI (Relative Strength Index) é um dos indicadores mais populares do trading — mas sofre de um problema: dispara sinais em mercados laterais. O ADX (Average Directional Index) mede a força da tendência, não a direção. Combinar os dois parece promissor: operar o RSI só quando o ADX confirma que há tendência.

Neste exemplo, você vai descobrir que **o filtro reduz taxas — mas não cria edge**.

---

## O que são RSI e ADX?

| Indicador | O que mede | Região | Interpretação |
|-----------|-----------|--------|---------------|
| **RSI** | Momentum (velocidade das variações de preço) | 0–100 | < 30 = oversold · > 70 = overbought |
| **ADX** | Força da tendência (independente de direção) | 0–100 | > 25 = tendência forte · < 20 = mercado lateral |

### Por que combinar?

- **RSI sozinho:** gera sinais de reversão em qualquer mercado — incluindo laterais, onde esses sinais são falsos.
- **ADX sozinho:** não diz se comprar ou vender — só diz se o mercado está em tendência.
- **Combinação:** usar RSI como gatilho e ADX como filtro. só operar quando RSI está em extremos **E** ADX confirma tendência forte.

A ideia é reduzir whipsaws (sinais falsos em mercado lateral). A pergunta é: essa redução compensa o custo de perder sinais bons?

---

## Passo 1 — Buscar a série

**Chat (IA):**
> Busque BTCUSDT em 15m da Binance

**Tool call:**
```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

A série fica disponível em `ct://series/binance/BTCUSDT/15m` com 1.724 candles (~18 dias).

---

## Passo 2 — Backtest RSI puro (sem fee)

Antes de adicionar o filtro, vamos ver o RSI sozinho.

**Chat (IA):**
> Rode um backtest: compre quando RSI < 30, venda quando RSI > 70, senão zerado. Capital 1000, sem fee.

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "rsi_puro_nofee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "num_trades": 135,
  "pnl_total": 1496.02,
  "pnl_bruto": 1496.02,
  "fees_totais": 0,
  "sharpe": 0.033,
  "sortino": 0.049,
  "win_rate": 0.7333,
  "profit_factor": 1.228,
  "avg_win": 81.29,
  "avg_loss": -181.99,
  "payoff_ratio": 0.447,
  "drawdown_max": 70.07,
  "exposicao": 0.2123,
  "num_long": 56,
  "num_short": 79,
  "max_wins_seguidos": 15,
  "max_perdas_seguidas": 3
}
```

### O alarde dos 135 trades

Sem taxas, o RSI puro é lucrativo: **+$1.496** com 73,3% de win rate. Parece excelente.

Mas olhe mais de perto:

- **Payoff ratio = 0,447:** a média de ganho ($81) é menos da metade da média de perda (-$182). A estratégia ganha frequentemente mas perde grande.
- **Drawdown de 70%:** em algum ponto o equity caiu 70% do pico.
- **135 trades em 1.724 candles:** um trade a cada ~13 candles. É muita rotatividade.

A pergunta crítica: quanto desse $1.496 sobrevive às taxas?

---

## Passo 3 — RSI puro com fee 0,1%

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_puro_fee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "num_trades": 135,
  "pnl_total": -15833.95,
  "pnl_bruto": 1496.02,
  "fees_totais": 17329.97,
  "sharpe": 0.040,
  "win_rate": 0.0963,
  "profit_factor": 0.099
}
```

O edge bruto de **+$1.496** é aniquilado por **$17.330** em taxas. A razão fee/edge é **11,6:1**.

Com 135 trades a ~$128 round-trip:

```
$128 × 135 = $17.280 ≈ $17.330 ✓
```

**O RSI puro é o pior caso para taxas:** muitos trades (135), cada um com edge pequeno. O custo de transação é 11× maior que todo o edge bruto.

---

## Passo 4 — O filtro ADX

A teoria: se filtrarmos sinais de RSI em mercados sem tendência (ADX baixo), reduzimos o número de trades e o whipsaw.

### A pegadinha do ADX como receita

ADX retorna um mapa com múltiplas colunas (`adx`, `plus_di`, `minus_di`), não uma série única. Por isso, **não funciona como `indicadores_receitas`** — precisa ser materializado via pipeline.

**Tool call (pipeline):**
```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_adx_rsi",
    "output": "$concat",
    "steps": [
      { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
      { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
      {
        "id": "concat",
        "op": "compose",
        "columns": [
          { "source": "$adx", "source_column": "adx", "as_column": "adx" },
          { "source": "$rsi", "source_column": "rsi", "as_column": "rsi" }
        ]
      }
    ]
  }
}
```

**Retorno real:**
```json
{
  "uri": "ct://derived/btc_adx_rsi",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1724,
  "first_ts": 1784960100,
  "last_ts": 1786510800,
  "steps_executed": 3
}
```

A pipeline produz uma série derivada com as colunas `adx` e `rsi` alinhadas barra a barra. O backtest usa `indicadores` (URI) em vez de `indicadores_receitas`.

---

## Passo 5 — Backtest RSI+ADX (sem fee)

**Estratégia:** comprar quando RSI < 30 **E** ADX > 25; vender quando RSI > 70 **E** ADX > 25.

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_adx_rsi",
    "capital_inicial": 1000,
    "fee_pct": 0,
    "nome": "rsi_adx_nofee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "num_trades": 72,
  "pnl_total": -146.79,
  "pnl_bruto": -146.79,
  "fees_totais": 0,
  "sharpe": 0.026,
  "win_rate": 0.6389,
  "profit_factor": 0.963,
  "drawdown_max": 79.06,
  "exposicao": 0.1288
}
```

O filtro ADX reduziu os trades de 135 para 72 (–47%), mas o PnL bruto foi de +$1.496 para **–$146,79**. Ou seja: o filtro eliminou mais edge do que whipsaw.

---

## Passo 6 — Backtest RSI+ADX (com fee 0,1%)

**Tool call:**
```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_adx_rsi",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_adx_fee"
  }
}
```

**Retorno real (resumo):**
```json
{
  "num_trades": 72,
  "pnl_total": -9380.84,
  "pnl_bruto": -146.79,
  "fees_totais": 9234.05,
  "sharpe": -0.013,
  "win_rate": 0.0972,
  "profit_factor": 0.066,
  "drawdown_max": 9.49,
  "exposicao": 0.1288,
  "avg_win": 94.43,
  "avg_loss": -154.49,
  "payoff_ratio": 0.611
}
```

Com fee, as perdas totais são $9.381 — menos que o RSI puro com fee ($15.834) porque há menos trades, mas ainda catastrófico.

---

## Passo 7 — Variações de threshold ADX

Será que um threshold diferente de ADX funciona melhor? Vamos testar 20 (mais permissivo) e 30 (mais restritivo).

### ADX > 20 (mais permissivo, mais trades)

**Estratégia:** `ind["adx"][0] > 20.0` em vez de `> 25.0`

**Retorno real (resumo):**
```json
{
  "num_trades": 95,
  "pnl_total": -12101.97,
  "pnl_bruto": 88.40,
  "fees_totais": 12190.37,
  "sharpe": 0.023,
  "win_rate": 0.0947,
  "profit_factor": 0.059,
  "drawdown_max": 12.21
}
```

### ADX > 30 (mais restritivo, menos trades)

**Estratégia:** `ind["adx"][0] > 30.0`

**Retorno real (resumo):**
```json
{
  "num_trades": 53,
  "pnl_total": -5902.16,
  "pnl_bruto": 880.48,
  "fees_totais": 6782.64,
  "sharpe": 0.019,
  "win_rate": 0.1698,
  "profit_factor": 0.120,
  "drawdown_max": 6.01
}
```

---

## Tabela comparativa — 7 estratégias

| # | Estratégia | Trades | PnL Total | PnL Bruto | Fees | Sharpe | Win% | PF | Payoff | DD Max |
|---|-----------|--------|-----------|-----------|------|--------|------|----|--------|--------|
| 1 | RSI puro (sem fee) | 135 | +$1.496 | +$1.496 | $0 | 0,033 | 73,3% | 1,228 | 0,447 | 70,1% |
| 2 | RSI puro (fee 0,1%) | 135 | -$15.834 | +$1.496 | $17.330 | 0,040 | 9,6% | 0,099 | — | — |
| 3 | RSI+ADX>25 (sem fee) | 72 | -$147 | -$147 | $0 | 0,026 | 63,9% | 0,963 | — | 79,1% |
| 4 | RSI+ADX>25 (fee 0,1%) | 72 | -$9.381 | -$147 | $9.234 | -0,013 | 9,7% | 0,066 | 0,611 | 9,5% |
| 5 | RSI+ADX>20 (fee 0,1%) | 95 | -$12.102 | +$88 | $12.190 | 0,023 | 9,5% | 0,059 | — | 12,2% |
| 6 | RSI+ADX>30 (fee 0,1%) | 53 | -$5.902 | +$880 | $6.783 | 0,019 | 17,0% | 0,120 | — | 6,0% |
| 7 | Buy & Hold | 0 | -$282 | — | — | -0,024 | — | — | — | 1,3% |

### Análise por coluna

**Trades:** o filtro ADX reduz de 135 (RSI puro) para 72 (ADX>25), 95 (ADX>20) e 53 (ADX>30). Menos trades = menos taxas.

**PnL Bruto:** aqui está o problema. RSI puro tem +$1.496 bruto, mas o filtro ADX>25 reduz para **–$147**. O filtro destruiu praticamente todo o edge bruto. ADX>30 é melhor (+$880 bruto) mas ainda abaixo do RSI puro.

**Fees:** ADX>30 paga $6.783 em taxas (vs $17.330 do RSI puro). Redução de 61% nas taxas — mas como o edge bruto caiu de $1.496 para $880, a razão edge/fee piorou.

**Melhor variante com fee:** ADX>30 é a menos pior ($5.902 prejuízo vs $15.834 puro), porque tem menos trades e edge bruto positivo.

---

## O dilema do filtro

O resultado revela a tensão fundamental do filtragem:

```
                    RSI puro          ADX>25            ADX>30
Trades:               135               72                53
Edge bruto:        +$1.496           -$147            +$880
Fees (0.1%):       $17.330          $9.234            $6.783

Edge/trade:          $11,08           -$2,04            $16,61
Fee/trade:           $128,37          $128,25           $128,00
Razão edge/fee:      0,086            -0,016            0,130
```

A lição:

1. **O filtro reduziu trades** (de 135 para 53) — o que é bom para taxas.
2. **Mas o filtro também reduziu edge bruto** (de $1.496 para $880 no melhor caso) — o que é ruim para PnL.
3. **A razão edge/fee melhorou** (de 0,086 para 0,130 com ADX>30) mas ainda está uma ordem de grandeza abaixo de 1,0.
4. **Filtro não cria edge** — ele só remove sinais. Se alguns desses sinais eram bons, você perdeu edge.

A pergunta certa não é "o filtro melhora o win rate?" (sim, de 9,6% para 17,0% com fee) mas sim "o filtro melhora a razão edge/fee o suficiente para a estratégia ficar positiva?" (não — 0,13 << 1,0).

---

## Variações pra experimentar

| # | Variação | Como | Hipótese |
|---|----------|-----|----------|
| 1 | **RSI mais apertado** | Oversold 25, overbought 75 | Menos sinais, menos whipsaw |
| 2 | **RSI com EMA** | Trocar `rsi(close, 14)` por `rsi(ema(close, 14), 14)` | Suaviza o RSI, reduz rugas |
| 3 | **ADX com DI direcional** | Só compra se DI+ > DI− E ADX > 25 | Filtra reversões contra a tendência |
| 4 | **ATR para sizing** | `atr(high, low, close, 14)` e ajustar lote por vol | Reduz risco em alta volatilidade |
| 5 | **Timeframe maior** | Re-buscar com `interval: "1h"` | Menos trades, edge maior por trade |
| 6 | **RSI com saída por ATR** | Saída após X barras em vez de RSI > 70 | Evita esperar overbought para sair |
| 7 | **Combinar com SMA** | Só operar se close > SMA 200 (filtro de tendência de longo prazo) | Alinha com tendência macro |

---

## Próximos passos

- [Exemplo 03 — Backtest com lib `grupo`](./03-lib-grupo-backtest.pt.md) — Execução sofisticada com entradas escalonadas e stops OCO.
- [Exemplo 04 — Teste de sobrevivência](./04-teste-sobrevivencia.pt.md) — O piso de lado arbitrário.
- [Exemplo 08 — Microestrutura: TFI](./08-tfi-backtest.pt.md) — Indicadores de fluxo de ordens.
- [Exemplo 12 — Rhai inline](./12-rhai-inline-backtest.pt.md) — 8 estratégias clássicas comparadas.

---

> **Lição:** Um filtro que reduz trades sem melhorar a razão edge/fee é só cosmético. O ADX cortou os whipsaws — mas cortou junto os trades que geravam o edge. Filtrar é fácil; filtrar **certo** é difícil.

---

*Todos os resultados são saídas reais de uma sessão MCP do CT Lab em BTCUSDT 15m (1.724 candles). Nenhum valor é ilustrativo.*
