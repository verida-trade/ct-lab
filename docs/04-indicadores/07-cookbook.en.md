# Indicator Cookbook

> Ready-to-use recipes for the most common indicator composition patterns.

---

## 1. Crossover → signal {-1, 0, +1}

**Pipeline:**
```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "cross_signal",
  "output": "$signal",
  "steps": [
    { "id": "fast", "op": "sma", "source": "$anchor", "period": 9 },
    { "id": "slow", "op": "sma", "source": "$anchor", "period": 21 },
    { "id": "above", "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" },
    { "id": "below", "op": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_abaixo" },
    {
      "id": "signal",
      "op": "condicional",
      "condicao": "$above",
      "entao": { "escalar": 1.0 },
      "senao": { "fonte": "$below" },
      "coluna_saida": "signal"
    }
  ]
}
```

---

## 2. Z-score of an indicator

**Rhai:**
```rhai
let r = rsi(close, 14);
let m = avg(r);
let s = 2.0;
(r - m) / s
```

**Pipeline (real stddev):**
```json
{
  "steps": [
    { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
    { "id": "mean", "op": "estatistica_rolling", "source": "$rsi", "metodo": "smm", "periodo": 50 },
    { "id": "std", "op": "estatistica_rolling", "source": "$rsi", "metodo": "desvio_padrao", "periodo": 50 },
    { "id": "z", "op": "custom", "script": "(ent[\"rsi\"] - ent[\"mean\"]) / ent[\"std\"]", "entradas": [{"alias":"rsi","fonte":"$rsi"},{"alias":"mean","fonte":"$mean"},{"alias":"std","fonte":"$std"}], "coluna_saida": "zscore" }
  ]
}
```

---

## 3. Conditional signal (RSI < 30 AND ADX > 25)

**Pipeline:**
```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "rsi_adx_signal",
  "output": "$signal",
  "steps": [
    { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
    { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
    { "id": "rsi_low", "op": "condicional", "condicao": "$rsi", "coluna_condicao": "rsi", "entao": {"escalar": 0.0}, "senao": {"escalar": 1.0}, "coluna_saida": "oversold" },
    { "id": "adx_diff", "op": "combinar_aritmetica", "operacao": "subtrair", "parcelas": [{"fonte":"$adx","coluna":"adx"},{"escalar": 25.0}], "coluna_saida": "diff" },
    { "id": "adx_strong", "op": "condicional", "condicao": "$adx_diff", "coluna_condicao": "diff", "entao": {"escalar": 1.0}, "senao": {"escalar": 0.0}, "coluna_saida": "adx_strong" },
    { "id": "signal", "op": "combinar_aritmetica", "operacao": "multiplicar", "parcelas": [{"fonte":"$rsi_low","coluna":"oversold"},{"fonte":"$adx_strong","coluna":"adx_strong"}], "coluna_saida": "signal" }
  ]
}
```

---

## 4. Price × indicator divergence

Detects when price makes higher high but RSI makes lower high (bearish divergence):

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "bearish_divergence",
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

## 5. ADX + DI filter (strong trend only)

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "adx_filter",
  "output": "$filter",
  "steps": [
    { "id": "adx", "op": "adx", "source": "$anchor", "period": 14 },
    { "id": "di_pos_gt_neg", "op": "comparar", "esquerda": "$adx", "coluna_esquerda": "plus_di", "direita": "$adx", "coluna_direita": "minus_di", "operador": "maior" },
    { "id": "adx_diff", "op": "combinar_aritmetica", "operacao": "subtrair", "parcelas": [{"fonte":"$adx","coluna":"adx"},{"escalar": 25.0}], "coluna_saida": "diff" },
    { "id": "adx_gt_25", "op": "condicional", "condicao": "$adx_diff", "coluna_condicao": "diff", "entao": {"escalar": 1.0}, "senao": {"escalar": 0.0}, "coluna_saida": "forte" },
    { "id": "filter", "op": "combinar_aritmetica", "operacao": "multiplicar", "parcelas": [{"fonte":"$di_pos_gt_neg","coluna":"sinal"},{"fonte":"$adx_gt_25","coluna":"forte"}], "coluna_saida": "trend_long" }
  ]
}
```

---

## 6. Normalized composite of N indicators

Sum 3 indicators normalized to [-1, +1]:

```rhai
let r = rsi(close, 14) - 50.0;
let m = cmo(close, 14);
let k = stochastic(high, low, close, 14, 3)["k"] - 50.0;
(r + m + k) / 3.0
```

---

## 7. Vol-adjusted return

```rhai
let ret = close - close[1];
let vol = atr(high, low, close, 14);
ret / vol
```

---

## 8. Adaptive envelope (Bollinger + ATR)

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "adaptive_envelope",
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

> Next: [Measuring structure](./08-medir-estrutura.en.md) · [Custom indicators](./09-indicadores-custom.en.md)
