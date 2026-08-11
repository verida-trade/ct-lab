# Regime Transition Model

> `ct_regime` + HMM + `aplicar_modelo` — detect and predict market regime.

## `ct_regime`

Premium indicator producing 9 dimensionless columns:

| Column | What it measures |
|---|---|
| `dir` | Movement direction |
| `vol` | Volatility regime |
| `vlm` | Volume regime |
| `fase` | Cycle phase |
| `progresso` | Regime progress |
| `range_ativo` | Range active (0/1) |
| `range_idade` | Range age |
| `amp_rel` | Relative amplitude |
| `vlm_rel` | Relative volume |

## Install regime models

Pre-trained models (gbdt + count matrix as stdlib lookup):

```json
{ "name": "instalar_modelos_regime", "arguments": {} }
```

## Use in prompt

The premium prompt `regime` guides the cycle: train regime → install → apply.

## Serving

```json
{
  "name": "aplicar_modelo",
  "arguments": { "modelo": "ct://models/regime_model", "fonte": "ct://series/binance/BTCUSDT/15m", "probas": true }
}
```

---

> Next: [Custom transform](./18-transformar-custom.en.md)
