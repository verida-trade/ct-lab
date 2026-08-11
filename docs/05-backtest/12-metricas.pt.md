# Métricas de Backtest

> Referência completa das métricas retornadas por `ct_backtest`.

| Métrica | O que mede | Fórmula/Fonte |
|---|---|---|
| `pnl_total` | Lucro/prejuízo líquido após taxas | `pnl_bruto - fees_totais` |
| `pnl_bruto` | Lucro/prejuízo antes de taxas | Soma dos P&L de cada trade |
| `fees_totais` | Total pago em taxas | Soma de `|notional| × fee_pct` por trade |
| `retorno_total` | Retorno % total | `pnl_total / capital_inicial × 100` |
| `retorno_anualizado` | Retorno % anualizado | Baseado no período da série |
| `retorno_medio_periodo` | Retorno médio por barra | |
| `drawdown_max` | Maior queda do equity (%) | Pico → vale máximo |
| `drawdown_duracao_max` | Maior duração do drawdown (barras) | |
| `volatilidade` | Volatilidade dos retornos (anualizada) | `std(retornos) × √(barras/ano)` |
| `downside_dev` | Desvio dos retornos negativos | |
| `sharpe` | Retorno ajustado ao risco | `(retorno - rf) / volatilidade` |
| `sortino` | Sharpe com downside deviation | `(retorno - rf) / downside_dev` |
| `calmar` | Retorno / drawdown_max | |
| `win_rate` | % de trades vencedores | `num_wins / num_trades` |
| `profit_factor` | Ganhos ÷ perdas | `soma_ganhos / |soma_perdas|` |
| `num_wins` | Número de trades vencedores | |
| `num_losses` | Número de trades perdedores | |
| `avg_win` | Ganho médio por trade vencedor | |
| `avg_loss` | Perda média por trade perdedor | |
| `expectancy` | Valor esperado por trade | |
| `payoff_ratio` | `avg_win / |avg_loss|` | |
| `melhor_trade` | Maior ganho em um trade | |
| `pior_trade` | Maior perda em um trade | |
| `max_wins_seguidos` | Máximo de wins consecutivos | |
| `max_perdas_seguidas` | Máximo de perdas consecutivas | |
| `duracao_media_trade` | Duração média de um trade (barras) | |
| `exposicao` | % do tempo com posição aberta | |
| `num_long` | Número de trades long | |
| `num_short` | Número de trades short | |
| `win_rate_long` | Win rate dos trades long | |
| `win_rate_short` | Win rate dos trades short | |
| `equity_uri` | URI da série de equity | `ct://derived/<nome>_equity` |
| `trades` | Lista de trades individuais | |

## Equity curve

A série de equity está disponível em `equity_uri`. Leia via resource template:

```
ct://derived/<nome>_equity/tail/100
```

---

> Voltar para: [README](./README.pt.md)
