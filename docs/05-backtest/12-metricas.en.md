# Backtest Metrics

> Complete reference of metrics returned by `ct_backtest`.

| Metric | What it measures | Formula/Source |
|---|---|---|
| `pnl_total` | Net profit/loss after fees | `pnl_bruto - fees_totais` |
| `pnl_bruto` | Profit/loss before fees | Sum of each trade's P&L |
| `fees_totais` | Total fees paid | Sum of `|notional| × fee_pct` per trade |
| `retorno_total` | Total return % | `pnl_total / capital_inicial × 100` |
| `retorno_anualizado` | Annualized return % | Based on series period |
| `drawdown_max` | Largest equity decline (%) | Peak → max valley |
| `drawdown_duracao_max` | Longest drawdown duration (bars) | |
| `volatilidade` | Return volatility (annualized) | `std(returns) × √(bars/year)` |
| `downside_dev` | Downside deviation | |
| `sharpe` | Risk-adjusted return | `(return - rf) / volatility` |
| `sortino` | Sharpe with downside deviation | `(return - rf) / downside_dev` |
| `calmar` | Return / max drawdown | |
| `win_rate` | % of winning trades | `num_wins / num_trades` |
| `profit_factor` | Gains ÷ losses | `sum_gains / |sum_losses|` |
| `num_wins` | Number of winning trades | |
| `num_losses` | Number of losing trades | |
| `avg_win` | Average gain per winning trade | |
| `avg_loss` | Average loss per losing trade | |
| `expectancy` | Expected value per trade | |
| `payoff_ratio` | `avg_win / |avg_loss|` | |
| `melhor_trade` | Best single trade gain | |
| `pior_trade` | Worst single trade loss | |
| `max_wins_seguidos` | Max consecutive wins | |
| `max_perdas_seguidas` | Max consecutive losses | |
| `duracao_media_trade` | Average trade duration (bars) | |
| `exposicao` | % of time with open position | |
| `num_long` | Number of long trades | |
| `num_short` | Number of short trades | |
| `win_rate_long` | Win rate of long trades | |
| `win_rate_short` | Win rate of short trades | |
| `equity_uri` | Equity series URI | `ct://derived/<name>_equity` |
| `trades` | List of individual trades | |

## Equity curve

The equity series is available at `equity_uri`. Read via resource template:

```
ct://derived/<name>_equity/tail/100
```

---

> Back to: [README](./README.en.md)
