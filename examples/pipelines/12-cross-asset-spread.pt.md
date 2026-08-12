# Receita 12 — Cross-Asset Spread (BTC vs ETH)

> **Nível:** Intermediário
> **Pré-requisitos:** Receitas 01–04 (séries, backtest, indicadores), familiarity with `compor_serie`

---

## O conceito

- **Cross-asset relative value**: em vez de olhar o preço absoluto do BTC, usamos a *razão* entre dois ativos correlacionados (BTC/ETH) como sinal.
- A razão BTC/ETH mede quanto BTC vale em termos de ETH. Quando a razão sobe, BTC está relativamente mais forte; quando cai, ETH outperforma.
- A hipótese de **momentum**: se a razão está subindo, BTC deve continuar mais forte → ficar longo BTC.
- A hipótese de **reversão (mean-reversion)**: se a razão subiu demais, tende a reverter → ficar longo BTC quando a razão cai (apostando que BTC vai recuperar).
- Esta receita mostra como buscar duas séries, compô-las numa série derivada, e usar os valores como indicador no backtest.

---

## Passo 1 — Buscar ambas as séries

O BTC já estava em cache (`ct://series/binance/BTCUSDT/15m`, 1724 candles). Para o ETH, usamos `buscar_binance_historico` alinhando o intervalo de tempo com o BTC via `since`/`until` (unix seconds como strings):

```json
{
  "name": "buscar_binance_historico",
  "arguments": {
    "symbol": "ETHUSDT",
    "interval": "15m",
    "since": "1784960100",
    "until": "1786510800"
  }
}
```

**Resultado:**

| Campo | Valor |
|---|---|
| uri | `ct://series/binance/ETHUSDT/15m` |
| row_count | 1724 |
| first_ts | 1784960100 |
| last_ts | 1786510800 |
| chunks | 2 |
| concurrency | 8 |

> Ambas as séries têm 1724 candles com o mesmo intervalo temporal — essencial para o `compor_serie` funcionar corretamente.

---

## Passo 2 — Compor série derivada (BTC + ETH)

Usamos `compor_serie` com BTC como **anchor** e ETH como coluna adicional:

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "btc_eth_v3",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth" }
    ]
  }
}
```

**Resultado:**

| Campo | Valor |
|---|---|
| uri | `ct://derived/btc_eth_v3` |
| row_count | 1724 |
| first_ts | 1784960100 |
| last_ts | 1786510800 |

> A série derivada tem colunas `btc` e `eth` alinhadas por timestamp. O backtest vai usar `indicadores: "ct://derived/btc_eth_v3"` e a estratégia lê `ind["btc"]` e `ind["eth"]`.

---

## Passo 3 — Backtest momentum (razão subindo → long BTC)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "let ratio = ind[\"btc\"][0] / ind[\"eth\"][0]; let ratio_prev = ind[\"btc\"][5] / ind[\"eth\"][5]; if ratio > ratio_prev { comprado(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_eth_v3",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "cross_mom_fee"
  }
}
```

**Lógica:** `ratio` = BTC/ETH agora; `ratio_prev` = BTC/ETH de 5 candles atrás. Se a razão subiu, fica long BTC.

---

## Passo 4 — Backtest reversão (razão caindo → long BTC)

Mesma estrutura, invertendo a condição:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "let ratio = ind[\"btc\"][0] / ind[\"eth\"][0]; let ratio_prev = ind[\"btc\"][5] / ind[\"eth\"][5]; if ratio < ratio_prev { comprado(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_eth_v3",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "cross_rev_fee"
  }
}
```

**Lógica:** se a razão caiu (BTC enfraqueceu relativamente), fica long BTC apostando na reversão (mean-reversion).

---

## Resultado real

| Estratégia | Trades | P&L Líquido | Bruto | Fees | Sharpe | Win% | PF | DD% |
|---|---|---|---|---|---|---|---|---|
| Momentum (fee) | 198 | −$29,204.00 | −$3,765.62 | $25,438.38 | 0.043 | 14.6% | 0.105 | 29.2% |
| Momentum (nofee) | 198 | −$3,765.62 | −$3,765.62 | — | −0.010 | 46.5% | 0.727 | — |
| Reversal (fee) | 197 | −$21,972.75 | +$3,513.34 | $25,310.57 | 0.030 | 16.8% | 0.207 | 22.0% |
| Reversal (nofee) | 197 | +$3,465.37 | +$3,513.34 | — | 0.046 | 47.2% | 1.367 | — |
| Buy & Hold BTC | — | −$281.50 | — | — | −0.024 | — | — | 1.25% |

> Note: `exp=48.5%`, `avg_win=$117.54`, `avg_loss=−$192.97`, `payoff=0.609` para momentum; `num_long=198` (100% das trades).

---

## Interpretação

- **Reversal tem edge bruto positivo**: +$3,513 sem fees, PF=1.367. Mean-reversion na razão BTC/ETH funciona — quando BTC enfraquece vs ETH, tende a reverter.
- **Momentum não tem edge**: bruto negativo (−$3,766), PF=0.727. BTC outperformando ETH *não* prevê continuação —比率 subindo não é sinal de momentum.
- **O problema é o turnover**: 197–198 trades em ~1724 candles de 15m = trade a cada ~8.7 candles (~2h). Cada trade paga ~$130 de fee → $25k total em fees.
- **Reversal com fees é negativo**: $25k de fees consome o edge de $3.5k. Para ser lucrativo, precisa reduzir o turnover drasticamente.
- **Comparação com B&H**: Buy & Hold BTC perdeu apenas $281 (−0.28%) com DD de 1.25%. As estratégias churn muito mais e perdem muito mais.
- **Conclusão**: o sinal de cross-asset tem valor preditivo (reversão), mas a implementação em 15m com lookback curto gera churn insustentável. Precisa de filtro de volatilidade ou threshold maior.

---

## Variações

- **Lookback maior**: trocar `ind["btc"][5]` por `ind["btc"][20]` ou `ind["btc"][50]` — menos trades, menos fees, possivelmente melhor sina/noise ratio.
- **Filtro ATR**: só entrar quando `atr[0] > média(atr, 20)` — evita operar em mercado de baixa volatilidade onde o spread significa menos.
- **Long/short ambos**: когда razão sobe → long ETH/short BTC; quando cai → long BTC/short ETH. Captura o spread bilateral, mas requer margem.
- **Trocar o ativo**: usar `SOLUSDT` no lugar de ETH. SOL tem correlação menor com BTC, possivelmente mais edge relativo (mas também mais risco idiossincrático).
