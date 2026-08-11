# FAQ — Performance

## Muitas séries no cache

Se a estratégia usa muitos indicadores, o cache (100 séries premium) pode encher.

**Solução:** Use `indicadores_receitas` (inline) em vez de pré-materializar — as receitas são computadas on-the-fly e não ocupam slot no cache.

## Backtest lento

- Reduza `num_trades` simplificando a estratégia
- Use timeframes maiores (15m em vez de 1m)
- Indicadores inline (`indicadores_receitas`) são mais rápidos que pré-materializar + URI

## Coleta de microestrutura com alto volume

100 símbolos × ~6000 msg/s pico — considere reduzir o número de símbolos se a CPU/IO limitar.
