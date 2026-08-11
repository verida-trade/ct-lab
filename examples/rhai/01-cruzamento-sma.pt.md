# Receita 01 — Primeira Estratégia (Cruzamento de SMA)

> **Nível:** Iniciante · **Tempo:** 10 min · **Pré-requisitos:** [Instalação](../docs/01-instalacao/) completa

Esta receita mostra o fluxo completo: buscar uma série, calcular indicadores, rodar um backtest simples e interpretar os resultados.

---

## Passo 1 — Buscar a série

**Chat (IA):**
> Busque BTCUSDT em 15m da Binance

**Tool call (MCP):**
```json
{
  "method": "tools/call",
  "params": {
    "name": "buscar_binance",
    "arguments": { "symbol": "BTCUSDT", "interval": "15m" }
  }
}
```

**Retorno:**
```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "evicted_series": []
}
```

---

## Passo 2 — Calcular duas SMAs (curta e longa)

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
  "row_count": 1000,
  "valid_from_ts": 1784068200,
  "value_names": ["sma"],
  "latest": [63745.12]
}
```

---

## Passo 3 — Montar pipeline com cruzamento

Em vez de dois cálculos separados, podemos usar a pipeline declarativa para produzir uma série derivada com ambas as SMAs e o sinal de cruzamento:

**Chat (IA):**
> Monte uma pipeline que calcule SMA 9, SMA 21 e um sinal de cruzamento (+1 quando a curta cruza acima da longa, -1 quando cruza abaixo)

**Tool call:**
```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_cruzamento_sma",
    "output": "$sinal",
    "steps": [
      { "id": "sma9", "operacao": "sma", "source": "$anchor", "period": 9 },
      { "id": "sma21", "operacao": "sma", "source": "$anchor", "period": 21 },
      {
        "id": "cruz",
        "operacao": "comparar",
        "esquerda": "$sma9",
        "direita": "$sma21",
        "operador": "cruza_acima"
      },
      {
        "id": "cruz_abaixo",
        "operacao": "comparar",
        "esquerda": "$sma9",
        "direita": "$sma21",
        "operador": "cruza_abaixo"
      },
      {
        "id": "sinal",
        "operacao": "condicional",
        "condicao": "$cruz",
        "entao": { "escalar": 1.0 },
        "senao": {
          "fonte": "$cruz_abaixo",
          "coluna": "cruza_abaixo"
        },
        "coluna_saida": "sinal"
      }
    ]
  }
}
```

> **Nota:** A pipeline é declarativa — cada step referencia steps anteriores via `$id`. Só o `output` final é persistido. Consulte `ct://pipeline/catalog` para ver todos os ops disponíveis.

---

## Passo 4 — Backtest com a estratégia

**Chat (IA):**
> Rode um backtest: compre quando SMA 9 > SMA 21, 否则 zerado. Capital inicial 1000, fee 0.1%

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
    "nome": "receita_sma_cross"
  }
}
```

**Retorno (resumo):**
```json
{
  "uri": "ct://backtest/receita_sma_cross",
  "equity_uri": "ct://derived/receita_sma_cross_equity",
  "num_trades": 47,
  "pnl_total": -12.34,
  "pnl_bruto": 45.67,
  "fees_totais": 58.01,
  "retorno_total": -1.23,
  "sharpe": -0.05,
  "sortino": -0.06,
  "calmar": -0.03,
  "win_rate": 0.42,
  "profit_factor": 0.87,
  "drawdown_max": 15.2,
  "drawdown_duracao_max": 340,
  "volatilidade": 0.32,
  "num_long": 47,
  "num_short": 0
}
```

> **Nota:** Os valores acima são ilustrativos. O resultado real depende dos dados no momento da execução.

---

## Passo 5 — Interpretar os resultados

| Métrica | O que mede | Como interpretar |
|---|---|---|
| `pnl_total` | Lucro/prejuízo líquido após taxas | Negativo = estratégia perdeu dinheiro |
| `pnl_bruto` | Lucro/prejuízo antes de taxas | Compara com `pnl_total` pra ver impacto de fees |
| `fees_totais` | Total pago em taxas | Se >> pnl_bruto, o turnover está comendo o edge |
| `sharpe` | Retorno ajustado ao risco | > 1 = bom; < 0 = pior que ficar parado |
| `win_rate` | % de trades vencedores | Isoladamente não diz muito — depende do payoff |
| `profit_factor` | Ganhos ÷ perdas | > 1 = lucrativo; < 1 = deficitário |
| `drawdown_max` | Maior queda do equity (%) | Mede o risco máximo |
| `num_trades` | Quantidade de trades | Poucos trades = estatística não significativa |

### Pontos de atenção

1. **Fees importam:** compare `pnl_bruto` vs `pnl_total`. Se a diferença for grande, a estratégia opera demais (turnover alto).
2. **Sharpe negativo:** a estratégia performou pior que ficar em caixa. Não é necessariamente inútil — pode funcionar em outro regime ou ativo.
3. **Win rate baixo com profit_factor > 1:** a estratégia perde frequentemente mas ganha grande — pode ser válida.
4. **Poucos trades:** com < 30 trades, as métricas têm baixa significância estatística.

---

## Variações pra experimentar

- **Trocar os períodos:** SMA 5 + SMA 20, ou SMA 50 + SMA 200 (golden cross)
- **Adicionar filtro ADX:** só opera quando ADX > 25 (tendência forte)
- **Usar RSI:** comprar quando RSI < 30 e SMA9 > SMA21
- **Backtest com `indicadores` (URI):** materializar a pipeline primeiro e passar a URI em `indicadores` em vez de `indicadores_receitas`

---

## Próximos passos

- [Indicadores e Pipeline](../docs/04-indicadores/) — catálogo completo e receitas avançadas
- [Backtest e Estratégias](../docs/05-backtest/) — lib `grupo`,.trailing, teste de sobrevivência
- [Receita 02 — RSI com filtro ADX](./02-rsi-filtro-adx.pt.md)
