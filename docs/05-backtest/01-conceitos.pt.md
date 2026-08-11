# Conceitos de Backtest

> Como o motor de backtest do `ct-mcp-server` funciona — ciclo barra-a-barra, inputs e outputs.

## Visão geral

O `ct_backtest` simula a execução de uma estratégia barra-a-barra sobre uma série OHLCV. A cada barra, a estratégia recebe os dados até o momento atual e decide uma posição: comprado, vendido ou zerado.

```
Barra 1 → [OHLCV + indicadores] → estratégia decide → posição
Barra 2 → [OHLCV + indicadores] → estratégia decide → posição
...
Barra N → resultado: equity curve, métricas, trades
```

---

## inputs da estratégia

A cada barra, a estratégia recebe:

| Variável | Tipo | O que é |
|---|---|---|
| `open[0]` | f64 | Open da barra atual |
| `high[0]` | f64 | High da barra atual |
| `low[0]` | f64 | Low da barra atual |
| `close[0]` | f64 | Close da barra atual |
| `volume[0]` | f64 | Volume da barra atual |
| `close[1]` | f64 | Close da barra anterior |
| `posicao` | f64 | Posição atual (0=zerado, >0=comprado, <0=vendido) |
| `ind["alias"][0]` | f64 | Valor atual do indicador por alias |
| `par["nome"]` | f64 | Parâmetro da estratégia |

> `[0]` = barra atual, `[1]` = barra anterior, `[k]` = k barras atrás.

## Saída: Alvo

A estratégia retorna um "alvo" — a posição desejada:

| Função Rhai | Significado |
|---|---|
| `comprado(1.0)` | Ir longo com 1.0 de tamanho |
| `vendido(1.0)` | Ir short com 1.0 de tamanho |
| `zerado()` | Fechar posição (flat) |
| `decisao(...)` | Ordem customizada (avançado, via lib grupo) |

> **Atenção:** sempre use f64 (1.0) não int (1) nos argumentos de Rhai.

---

## Chamada da tool

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "exemplo"
  }
}
```

### Parâmetros

| Parâmetro | Tipo | Default | Descrição |
|---|---|---|---|
| `serie` | string | obrigatório | URI da série OHLCV |
| `estrategia_script` | string | — | Script Rhai inline |
| `estrategia` | string | — | URI de arquivo .rhai/.py (alternativa) |
| `indicadores` | string | — | URI de série derivada com indicadores |
| `indicadores_receitas` | object | — |_alias → receita Rhai inline |
| `parametros` | object | — | Parâmetros expostos como `par["nome"]` |
| `capital_inicial` | number | — | Capital inicial |
| `fee_pct` | number | 0 | Fee por trade (0.001 = 0.1%) |
| `nome` | string | auto | Nome/id do backtest |
| `instrumento` | object | — | tick, lote_min, lote_step |

---

## Retorno

```json
{
  "uri": "ct://backtest/exemplo",
  "equity_uri": "ct://derived/exemplo_equity",
  "num_trades": 47,
  "pnl_total": -12.34,
  "pnl_bruto": 45.67,
  "fees_totais": 58.01,
  "retorno_total": -1.23,
  "retorno_anualizado": -0.98,
  "drawdown_max": 15.2,
  "sharpe": -0.05,
  "sortino": -0.06,
  "calmar": -0.03,
  "win_rate": 0.42,
  "profit_factor": 0.87,
  "volatilidade": 0.32,
  "num_long": 47,
  "num_short": 0,
  "trades": [...]
}
```

---

## Indicadores no backtest

Há duas formas de passar indicadores para a estratégia:

### (a) URI de série derivada (pipeline materializada)

```json
{ "indicadores": "ct://derived/meus_indicadores" }
```

Todas as colunas da série derivada viram aliases: `ind["nome_coluna"][0]`.

### (b) Receitas inline

```json
{
  "indicadores_receitas": {
    "rsi14": { "receita": "rsi(close, 14)" },
    "sma9": { "receita": "sma(close, 9)" }
  }
}
```

A tool materializa cada receita sobre a série e injeta como `ind["alias"]`. A estratégia declara o que precisa, sem pré-materializar.

---

> Próximo: [Estratégia declarativa](./02-estrategia-declarativa.pt.md)
