# Serving — `aplicar_modelo`

> Reaplique um modelo treinado sem re-treinar. O modelo é artefato opaco em `ct://models/<name>`.

## Chamada

```json
{
  "name": "aplicar_modelo",
  "arguments": {
    "modelo": "ct://models/meu_modelo",
    "fonte": "ct://series/binance/BTCUSDT/15m",
    "probas": true
  }
}
```

| Parâmetro | Descrição |
|---|---|
| `modelo` | URI do modelo treinado |
| `fonte` | URI raw ou derivada (âncora = raw da cadeia) |
| `probas` | `true` → materializa probabilidade POR CLASSE (colunas `p_<classe>`) em vez da classe argmax |

## Retorno

A predição é materializada como série derivada alinhada ao timeframe da âncora:

```
ct://derived/<model_name>_pred
```

Com `probas: true`, gera colunas `p_0`, `p_1`, `p_2` (uma por classe).

## Reaplicação

O `aplicar_modelo` reconstrói o ambiente `uv` com as mesmas deps, carrega o modelo e re-aplica os mesmos ajustes (scalers, transformações) como no treino. **Não re-treina**.

---

> Próximo: [EDA](./12-eda.pt.md)
