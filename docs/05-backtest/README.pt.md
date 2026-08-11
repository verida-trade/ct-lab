# Backtest e Estratégias

> Como testar estratégias de trading no CT Lab — desde o backtest simples até a lib `grupo` com stop/trailing e o teste de sobrevivência.

## Documentos

| # | Documento | O que cobre |
|---|---|---|
| 1 | [Conceitos de backtest](./01-conceitos.pt.md) | Como `ct_backtest` funciona |
| 2 | [Estratégia declarativa](./02-estrategia-declarativa.pt.md) | Config de estratégia simples |
| 3 | [Estratégia em Rhai](./03-estrategia-rhai.pt.md) | Script Rhai completo com estado |
| 4 | [Estratégia em Python](./04-estrategia-python.pt.md) | Script Python via `uv` |
| 5 | [A lib `grupo` (seed CT)](./05-lib-grupo.pt.md) | Grupos de ordens: entradas + saídas OCO |
| 6 | [Fork da lib `grupo`](./06-fork-lib-grupo.pt.md) | `salvar_lib`, evoluir a execução |
| 7 | [Camada adaptativa](./07-camada-adaptativa.pt.md) | `alvo_pos`, `saidas_vivas`, breakeven |
| 8 | [Teste de sobrevivência (Grid)](./08-teste-sobrevivencia.pt.md) | `ct_testar_sobrevivencia` — veredito Σ≥0 |
| 9 | [Comparação de backtests](./09-comparacao.pt.md) | `ct_comparar`, `ct_buscar_backtests` |
| 10 | [Indicadores inline](./10-indicadores-inline.pt.md) | `indicadores_receitas` |
| 11 | [O simulador](./11-simulador.pt.md) | Motor de execução, OCO, fee |
| 12 | [Métricas](./12-metricas.pt.md) | Sharpe, Sortino, Calmar, drawdown |

> Versão em inglês: [README.en.md](./README.en.md)
