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
    { "id": "fast", "operacao": "sma", "source": "$anchor", "period": 9 },
    { "id": "slow", "operacao": "sma", "source": "$anchor", "period": 21 },
    { "id": "above", "operacao": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_acima" },
    { "id": "below", "operacao": "comparar", "esquerda": "$fast", "direita": "$slow", "operador": "cruza_abaixo" },
    {
      "id": "signal",
      "operacao": "condicional",
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
    { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
    { "id": "mean", "operacao": "estatistica_rolling", "source": "$rsi", "metodo": "smm", "periodo": 50 },
    { "id": "std", "operacao": "estatistica_rolling", "source": "$rsi", "metodo": "desvio_padrao", "periodo": 50 },
    { "id": "z", "operacao": "custom", "script": "(ent[\"rsi\"] - ent[\"mean\"]) / ent[\"std\"]", "entradas": [{"alias":"rsi","fonte":"$rsi"},{"alias":"mean","fonte":"$mean"},{"alias":"std","fonte":"$std"}], "coluna_saida": "zscore" }
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
    { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
    { "id": "adx", "operacao": "adx", "source": "$anchor" },
    { "id": "rsi_low", "operacao": "condicional", "condicao": "$rsi", "coluna_condicao": "rsi", "entao": {"escalar": 0.0}, "senao": {"escalar": 1.0}, "coluna_saida": "oversold" },
    { "id": "adx_strong", "operacao": "comparar", "esquerda": "$adx", "coluna_esquerda": "adx", "direita": {"escalar": 25.0}, "operador": "maior" },
    { "id": "signal", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$rsi_low"},{"fonte":"$adx_strong"}], "coluna_saida": "signal" }
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
    { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
    { "id": "price_hh", "operacao": "comparar", "esquerda": "$anchor", "coluna_esquerda": "high", "direita": "$anchor", "coluna_direita": "close", "operador": "maior" },
    { "id": "rsi_lh", "operacao": "comparar", "esquerda": "$rsi", "direita": "$rsi", "operador": "menor" },
    { "id": "div", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$price_hh"},{"fonte":"$rsi_lh"}], "coluna_saida": "divergence" }
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
    { "id": "adx", "operacao": "adx", "source": "$anchor" },
    { "id": "di_pos_gt_neg", "operacao": "comparar", "esquerda": "$adx", "coluna_esquerda": "di_plus", "direita": "$adx", "coluna_direita": "di_minus", "operador": "maior" },
    { "id": "adx_gt_25", "operacao": "comparar", "esquerda": "$adx", "coluna_esquerda": "adx", "direita": {"escalar": 25.0}, "operador": "maior" },
    { "id": "filter", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$di_pos_gt_neg"},{"fonte":"$adx_gt_25"}], "coluna_saida": "trend_long" }
  ]
}
```

---

## 6. Normalized composite of N indicators

Sum 3 indicators normalized to [-1, +1]:

```rhai
let r = rsi(close, 14) - 50.0;
let m = cmo(close, 14);
let k = stochastic(high, low, close)["k"] - 50.0;
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
    { "id": "atr", "operacao": "atr", "source": "$anchor", "period": 14 },
    { "id": "sma", "operacao": "sma", "source": "$anchor", "period": 20 },
    { "id": "band_width", "operacao": "combinar_aritmetica", "operador": "multiplicar", "parcelas": [{"fonte":"$atr"},{"escalar": 2.0}], "coluna_saida": "bw" },
    { "id": "upper", "operacao": "combinar_aritmetica", "operador": "somar", "parcelas": [{"fonte":"$sma"},{"fonte":"$band_width","coluna":"bw"}], "coluna_saida": "upper" }
  ]
}
```

---

> Next: [Measuring structure](./08-medir-estrutura.en.md) · [Custom indicators](./09-indicadores-custom.en.md)
