# Recipe 01 — First Strategy (SMA Crossover)

> **Level:** Beginner · **Time:** 10 min · **Prerequisites:** [Installation](../docs/01-instalacao/) complete

This recipe shows the full workflow: fetch a series, compute indicators, run a backtest, and interpret results.

---

## Step 1 — Fetch the series

**AI Chat:**
> Fetch BTCUSDT at 15m from Binance

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

**Return:**
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

## Step 2 — Compute two SMAs (short and long)

**AI Chat:**
> Calculate SMA 9 and SMA 21 on BTCUSDT 15m

**Tool calls:**
```json
[
  { "name": "sma", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 9, "name": "btc_sma9" } },
  { "name": "sma", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 21, "name": "btc_sma21" } }
]
```

**Return (SMA 9):**
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

## Step 3 — Build a crossover pipeline

Instead of two separate calculations, use the declarative pipeline to produce a derived series with both SMAs and the crossover signal:

**AI Chat:**
> Build a pipeline that computes SMA 9, SMA 21, and a crossover signal (+1 when the short crosses above the long, -1 when it crosses below)

**Tool call:**
```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "btc_sma_crossover",
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

> **Note:** The pipeline is declarative — each step references prior steps via `$id`. Only the `output` step is persisted. Check `ct://pipeline/catalog` for all available ops.

---

## Step 4 — Run the backtest

**AI Chat:**
> Run a backtest: go long when SMA 9 > SMA 21, otherwise flat. Initial capital 1000, fee 0.1%

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
    "nome": "recipe_sma_cross"
  }
}
```

**Return (summary):**
```json
{
  "uri": "ct://backtest/recipe_sma_cross",
  "equity_uri": "ct://derived/recipe_sma_cross_equity",
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

> **Note:** Values above are illustrative. Actual results depend on the data at execution time.

---

## Step 5 — Interpret results

| Metric | What it measures | How to interpret |
|---|---|---|
| `pnl_total` | Net profit/loss after fees | Negative = strategy lost money |
| `pnl_bruto` | Profit/loss before fees | Compare with `pnl_total` to see fee impact |
| `fees_totais` | Total fees paid | If >> pnl_bruto, turnover is eating the edge |
| `sharpe` | Risk-adjusted return | > 1 = good; < 0 = worse than doing nothing |
| `win_rate` | % of winning trades | Alone doesn't say much — depends on payoff |
| `profit_factor` | Gains ÷ losses | > 1 = profitable; < 1 = losing |
| `drawdown_max` | Largest equity decline (%) | Measures maximum risk |
| `num_trades` | Number of trades | Few trades = not statistically significant |

### What to watch for

1. **Fees matter:** compare `pnl_bruto` vs `pnl_total`. A large gap means the strategy trades too often (high turnover).
2. **Negative Sharpe:** the strategy underperformed holding cash. Not necessarily useless — may work in a different regime or asset.
3. **Low win rate with profit_factor > 1:** the strategy loses often but wins big — can be valid.
4. **Few trades:** with < 30 trades, metrics have low statistical significance.

---

## Variations to try

- **Change periods:** SMA 5 + SMA 20, or SMA 50 + SMA 200 (golden cross)
- **Add ADX filter:** only trade when ADX > 25 (strong trend)
- **Use RSI:** buy when RSI < 30 AND SMA9 > SMA21
- **Backtest with `indicadores` (URI):** materialize the pipeline first and pass the URI in `indicadores` instead of `indicadores_receitas`

---

## Next steps

- [Indicators & Pipeline](../docs/04-indicadores/) — full catalog and advanced recipes
- [Backtest & Strategies](../docs/05-backtest/) — `grupo` lib, trailing, survival test
- [Recipe 02 — RSI with ADX filter](./02-rsi-filtro-adx.en.md)
