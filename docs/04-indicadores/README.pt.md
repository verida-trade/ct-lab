# Indicadores e Pipeline

> Como calcular, combinar e materializar indicadores técnicos no CT Lab.

Esta seção cobre desde o indicador individual mais simples até pipelines declarativos multi-step, Rhai vetorizado e indicadores premium da família CT.

## Documentos

| # | Documento | O que cobre |
|---|---|---|
| 1 | [Indicadores built-in (36 públicos)](./01-indicadores-publicos.pt.md) | Catálogo completo: RSI, MACD, Bollinger, ATR, OBV, etc. — como chamar cada `compute_*` |
| 2 | [Indicadores premium (17 CT)](./02-indicadores-premium.pt.md) | Família CT: `ct_bop`, `ct_tfi`, `ct_candle`, `ct_swing`, `ct_range`, `ct_tendencia`, etc. |
| 3 | [Pipeline declarativo (DAG)](./03-pipeline-declarativo.pt.md) | `montar_pipeline_indicadores` — steps encadeados via `$id`/`$anchor`, ops declarativas |
| 4 | [Ops declarativas](./04-ops-declarativas.pt.md) | `combinar_aritmetica`, `comparar`, `condicional`, `transformar`, `estatistica_rolling`, `compose`, `custom` |
| 5 | [Rhai vetorizado]((./05-rhai-vetorizado.pt.md) | `materializar_indicador` — tipo `Serie`, operadores, 45+ indicadores como funções série→série |
| 6 | [Compose cross-asset](./06-compose-cross-asset.pt.md) | Juntar N séries por timestamp, razão/spread entre ativos |
| 7 | [Cookbook de receitas](./07-cookbook.pt.md) | Cruzamento→{-1,0,1}, z-score, sinal condicional, divergência, filtro ADX |
| 8 | [Medir estrutura](./08-medir-estrutura.pt.md) | `ct_medir_estrutura` — variance ratio + curtose por escala/regime |
| 9 | [Indicadores custom](./09-indicadores-custom.pt.md) | Step `custom` com script Rhai ou Python inline/uri na pipeline |
| 10 | [Referência técnica — fórmulas](./10-formulas-publicos.pt.md) | Definição matemática dos 36 indicadores públicos (sem viés de interpretação) |

---

> Versão em inglês: [README.en.md](./README.en.md)
