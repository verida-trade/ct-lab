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
    { "id": "fast", "operacao": "sma", "source": "$anchor", "period": 9 },
    { "id": "slow", "operacao": "sma", "source": "$anchor", "period": 21 },
    { "id": "acima", "operacao": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" },
    { "id": "abaixo", "operacao": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_abaixo" },
    {
      "id": "sinal",
      "operacao": "condicional",
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
    { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
    { "id": "media", "operacao": "estatistica_rolling", "source": "$rsi", "metodo": "smm", "periodo": 50 },
    { "id": "desvio", "operacao": "estatistica_rolling", "source": "$rsi", "metodo": "desvio_padrao", "periodo": 50 },
    { "id": "z", "operacao": "custom", "script": "(ent[\"rsi\"] - ent[\"media\"]) / ent[\"desvio\"]", "entradas": [{"alias":"rsi","fonte":"$rsi"},{"alias":"media","fonte":"$media"},{"alias":"desvio","fonte":"$desvio"}], "coluna_saida": "zscore" }
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
    { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
    { "id": "adx", "operacao": "adx", "source": "$anchor" },
    { "id": "rsi_baixo", "operacao": "condicional", "condicao": "$rsi", "coluna_condicao": "rsi", "entao": {"escalar": 0.0}, "senao": {"escalar": 1.0}, "coluna_saida": "rsi_oversold" },
    { "id": "adx_forte", "operacao": "comparar", "esquerda": "$adx", "coluna_esquerda": "adx", "direita": {"escalar": 25.0}, "operador": "maior" },
    { "id": "sinal", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$rsi_baixo"},{"fonte":"$adx_forte"}], "coluna_saida": "sinal" }
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
    { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
    { "id": "price_hh", "operacao": "comparar", "esquerda": "$anchor", "coluna_esquerda": "high", "direita": "$anchor", "coluna_direita": "close", "operador": "maior" },
    { "id": "rsi_lh", "operacao": "comparar", "esquerda": "$rsi", "direita": "$rsi", "operador": "menor" },
    { "id": "div", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$price_hh"},{"fonte":"$rsi_lh"}], "coluna_saida": "divergence" }
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
    { "id": "adx", "operacao": "adx", "source": "$anchor" },
    { "id": "di_pos_gt_neg", "operacao": "comparar", "esquerda": "$adx", "coluna_esquerda": "di_plus", "direita": "$adx", "coluna_direita": "di_minus", "operador": "maior" },
    { "id": "adx_gt_25", "operacao": "comparar", "esquerda": "$adx", "coluna_esquerda": "adx", "direita": {"escalar": 25.0}, "operador": "maior" },
    { "id": "filtro", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$di_pos_gt_neg"},{"fonte":"$adx_gt_25"}], "coluna_saida": "tendencia_long" }
  ]
}
```

---

## 6. Composto normalizado de N indicadores

Soma 3 indicadores normalizados para [-1, +1]:

```rhai
let r = rsi(close, 14) - 50.0;
let m = cmo(close, 14);
let k = stochastic(high, low, close)["k"] - 50.0;
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
    { "id": "atr", "operacao": "atr", "source": "$anchor", "period": 14 },
    { "id": "sma", "operacao": "sma", "source": "$anchor", "period": 20 },
    { "id": "band_width", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$atr"},{"escalar": 2.0}], "coluna_saida": "bw" },
    { "id": "upper", "operacao": "combinar_aritmetica", "operador": "somar", "parcelas": [{"fonte":"$sma"},{"fonte":"$band_width","coluna":"bw"}], "coluna_saida": "upper" }
  ]
}
```

---

> Próximo: [Medir estrutura](./08-medir-estrutura.pt.md) · [Indicadores custom](./09-indicadores-custom.pt.md)
