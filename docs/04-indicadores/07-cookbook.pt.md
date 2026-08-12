# Cookbook de Indicadores

> Receitas prontas para os padrões mais comuns de composição de indicadores.

---

## 1. Cruzamento → sinal {-1, 0, +1}

**Pipeline:**
```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "sinal_cross",
  "output": "$sinal",
  "steps": [
    { "id": "fast", "op": "sma", "source": "$anchor", "period": 9 },
    { "id": "slow", "op": "sma", "source": "$anchor", "period": 21 },
    { "id": "acima", "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" },
    { "id": "abaixo", "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_abaixo" },
    {
      "id": "sinal",
      "op": "condicional",
      "condicao": "$acima",
      "entao": { "escalar": 1.0 },
      "senao": { "fonte": "$abaixo" },
      "coluna_saida": "sinal"
    }
  ]
}
```

---

## 2. Z-score de um indicador

**Rhai:**
```rhai
let r = rsi(close, 14);
let m = avg(r);
let s = 2.0;  // desvio aproximado; para desvio real use estatistica_rolling na pipeline
(r - m) / s
```

**Pipeline (desvio real):**
```json
{
  "steps": [
    { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
    { "id": "media", "op": "estatistica_rolling", "source": "$rsi", "metodo": "smm", "periodo": 50 },
    { "id": "desvio", "op": "estatistica_rolling", "source": "$rsi", "metodo": "desvio_padrao", "periodo": 50 },
    { "id": "z", "op": "custom", "script": "(ent[\"rsi\"] - ent[\"media\"]) / ent[\"desvio\"]", "entradas": [{"alias":"rsi","fonte":"$rsi"},{"alias":"media","fonte":"$media"},{"alias":"desvio","fonte":"$desvio"}], "coluna_saida": "zscore" }
  ]
}
```

---

## 3. Sinal condicional (RSI < 30 E ADX > 25)

**Pipeline:**
```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "sinal_rsi_adx",
  "output": "$sinal",
  "steps": [
    { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
    { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
    { "id": "rsi_baixo", "op": "condicional", "condicao": "$rsi", "coluna_condicao": "rsi", "entao": {"escalar": 0.0}, "senao": {"escalar": 1.0}, "coluna_saida": "rsi_oversold" },
    { "id": "adx_diff", "op": "combinar_aritmetica", "operacao": "subtrair", "parcelas": [{"fonte":"$adx","coluna":"adx"},{"escalar": 25.0}], "coluna_saida": "diff" },
    { "id": "adx_forte", "op": "condicional", "condicao": "$adx_diff", "coluna_condicao": "diff", "entao": {"escalar": 1.0}, "senao": {"escalar": 0.0}, "coluna_saida": "adx_forte" },
    { "id": "sinal", "op": "combinar_aritmetica", "operacao": "multiplicar", "parcelas": [{"fonte":"$rsi_baixo","coluna":"rsi_oversold"},{"fonte":"$adx_forte","coluna":"adx_forte"}], "coluna_saida": "sinal" }
  ]
}
```

---

## 4. Divergência preço × indicador

Detecta quando o preço faz high mais alto mas RSI faz high mais baixo (divergência bearish):

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "divergencia_bearish",
  "output": "$div",
  "steps": [
    { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
    { "id": "price_hh", "op": "comparar", "esquerda": "$anchor", "coluna_esquerda": "high", "direita": "$anchor", "coluna_direita": "close", "operador": "maior" },
    { "id": "rsi_lh", "op": "comparar", "esquerda": "$rsi", "direita": "$rsi", "operador": "menor" },
    { "id": "div", "op": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$price_hh"},{"fonte":"$rsi_lh"}], "coluna_saida": "divergence" }
  ]
}
```

---

## 5. Filtro ADX + DI (só tendência forte)

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "filtro_adx",
  "output": "$filtro",
  "steps": [
    { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
    { "id": "di_pos_gt_neg", "op": "comparar", "esquerda": "$adx", "coluna_esquerda": "plus_di", "direita": "$adx", "coluna_direita": "minus_di", "operador": "maior" },
    { "id": "adx_diff", "op": "combinar_aritmetica", "operacao": "subtrair", "parcelas": [{"fonte":"$adx","coluna":"adx"},{"escalar": 25.0}], "coluna_saida": "diff" },
    { "id": "adx_gt_25", "op": "condicional", "condicao": "$adx_diff", "coluna_condicao": "diff", "entao": {"escalar": 1.0}, "senao": {"escalar": 0.0}, "coluna_saida": "forte" },
    { "id": "filtro", "op": "combinar_aritmetica", "operacao": "multiplicar", "parcelas": [{"fonte":"$di_pos_gt_neg","coluna":"sinal"},{"fonte":"$adx_gt_25","coluna":"forte"}], "coluna_saida": "tendencia_long" }
  ]
}
```

---

## 6. Composto normalizado de N indicadores

Soma 3 indicadores normalizados para [-1, +1]:

```rhai
let r = rsi(close, 14) - 50.0;
let m = cmo(close, 14);
let k = stochastic(high, low, close, 14, 3)["k"] - 50.0;
(r + m + k) / 3.0
```

---

## 7. Retorno vol-ajustado

```rhai
let ret = close - close[1];
let vol = atr(high, low, close, 14);
ret / vol
```

---

## 8. Envelope adaptativo (Bollinger + ATR)

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "envelope_adapt",
  "output": "$upper",
  "steps": [
    { "id": "atr", "op": "atr", "source": "$anchor", "period": 14 },
    { "id": "sma", "op": "sma", "source": "$anchor", "period": 20 },
    { "id": "band_width", "op": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$atr"},{"escalar": 2.0}], "coluna_saida": "bw" },
    { "id": "upper", "op": "combinar_aritmetica", "operador": "somar", "parcelas": [{"fonte":"$sma"},{"fonte":"$band_width","coluna":"bw"}], "coluna_saida": "upper" }
  ]
}
```

---

> Próximo: [Medir estrutura](./08-medir-estrutura.pt.md) · [Indicadores custom](./09-indicadores-custom.pt.md)
