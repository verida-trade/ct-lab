# Backtest & Strategies

> How to test trading strategies in CT Lab — from simple backtests to the `grupo` lib with stop/trailing and the survival test.

## Documents

| # | Document | Covers |
|---|---|---|
| 1 | [Backtest concepts](./01-conceitos.en.md) | How `ct_backtest` works |
| 2 | [Declarative strategy](./02-estrategia-declarativa.en.md) | Simple strategy config |
| 3 | [Rhai strategy](./03-estrategia-rhai.en.md) | Full Rhai script with state |
| 4 | [Python strategy](./04-estrategia-python.en.md) | Python script via `uv` |
| 5 | [The `grupo` lib (CT seed)](./05-lib-grupo.en.md) | Order groups: entries + OCO exits |
| 6 | [Forking the `grupo` lib](./06-fork-lib-grupo.en.md) | `salvar_lib`, evolve execution |
| 7 | [Adaptive layer](./07-camada-adaptativa.en.md) | `alvo_pos`, `saidas_vivas`, breakeven |
| 8 | [Survival test (Grid)](./08-teste-sobrevivencia.en.md) | `ct_testar_sobrevivencia` — verdict Σ≥0 |
| 9 | [Comparing backtests](./09-comparacao.en.md) | `ct_comparar`, `ct_buscar_backtests` |
| 10 | [Inline indicators](./10-indicadores-inline.en.md) | `indicadores_receitas` |
| 11 | [The simulator](./11-simulador.en.md) | Execution engine, OCO, fees |
| 12 | [Metrics](./12-metricas.en.md) | Sharpe, Sortino, Calmar, drawdown |

> Portuguese version: [README.pt.md](./README.pt.md)
