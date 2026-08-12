# Indicators & Pipeline

> How to compute, combine, and materialize technical indicators in CT Lab.

This section covers everything from the simplest individual indicator to declarative multi-step pipelines, vectorized Rhai, and premium CT-family indicators.

## Documents

| # | Document | Covers |
|---|---|---|
| 1 | [Built-in indicators (36 public)](./01-indicadores-publicos.en.md) | Full catalog: RSI, MACD, Bollinger, ATR, OBV, etc. |
| 2 | [Premium indicators (17 CT)](./02-indicadores-premium.en.md) | CT family: `ct_bop`, `ct_tfi`, `ct_candle`, `ct_swing`, `ct_range`, `ct_tendencia`, etc. |
| 3 | [Declarative pipeline (DAG)](./03-pipeline-declarativo.en.md) | `montar_pipeline_indicadores` — chained steps via `$id`/`$anchor` |
| 4 | [Declarative ops](./04-ops-declarativas.en.md) | `combinar_aritmetica`, `comparar`, `condicional`, `transformar`, `estatistica_rolling`, `compose`, `custom` |
| 5 | [Vectorized Rhai](./05-rhai-vetorizado.en.md) | `materializar_indicador` — `Serie` type, operators, 45+ indicators as series→series functions |
| 6 | [Cross-asset compose](./06-compose-cross-asset.en.md) | Join N series by timestamp, ratio/spread between assets |
| 7 | [Recipe cookbook](./07-cookbook.en.md) | Crossover→{-1,0,1}, z-score, conditional signal, divergence, ADX filter |
| 8 | [Measuring structure](./08-medir-estrutura.en.md) | `ct_medir_estrutura` — variance ratio + kurtosis by scale/regime |
| 9 | [Custom indicators](./09-indicadores-custom.en.md) | `custom` step with Rhai or Python inline/uri in the pipeline |
| 10 | [Technical reference — formulas](./10-formulas-publicos.en.md) | Mathematical definition of all 36 public indicators (no signal interpretation) |

---

> Portuguese version: [README.pt.md](./README.pt.md)
