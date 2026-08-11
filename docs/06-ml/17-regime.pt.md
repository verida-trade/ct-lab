# Modelo de Transição de Regime

> `ct_regime` + HMM + `aplicar_modelo` — detecte e preva regime de mercado.

## `ct_regime`

Indicador premium que produz 9 colunas adimensionais:

| Coluna | O que mede |
|---|---|
| `dir` | Direção do movimento |
| `vol` | Regime de volatilidade |
| `vlm` | Regime de volume |
| `fase` | Fase do ciclo |
| `progresso` | Progresso no regime |
| `range_ativo` | Range ativo (0/1) |
| `range_idade` | Idade do range |
| `amp_rel` | Amplitude relativa |
| `vlm_rel` | Volume relativo |

## Instalar modelos de regime

Modelos pré-treinados (gbdt + matriz de contagem como lookup stdlib):

```json
{
  "name": "instalar_modelos_regime",
  "arguments": {}
}
```

## Usar no prompt

O prompt premium `regime` guia o ciclo: treinar regime → instalar → aplicar.

## Serving

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "ct://models/regime_model",
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "probas": true
  }
}
```

---

> Próximo: [Transformação custom](./18-transformar-custom.pt.md)
