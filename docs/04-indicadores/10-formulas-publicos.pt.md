# Referência Técnica — Fórmulas dos 36 Indicadores Públicos

> Definição matemática de cada indicador. Sem interpretação de sinal — apenas a formulação, parâmetros e saídas.

---

## Médias Móveis (6)

### SMA — Simple Moving Average

**Fórmula:**
```
SMA(n) = (1/n) × Σ close[i],  i = t−n+1 .. t
```

**Parâmetros:** `period: u8` (n) — default 20
**Saída:** 1 coluna — `sma`
**Warmup:** n barras

---

### EMA — Exponential Moving Average

**Fórmula:**
```
α = 2 / (n + 1)
EMA[t] = α × close[t] + (1 − α) × EMA[t−1]
EMA[0] = close[0]
```

**Parâmetros:** `period: u8` (n) — default 20
**Saída:** 1 coluna — `ema`
**Warmup:** ~2n barras (convergência)

---

### WMA — Weighted Moving Average

**Fórmula:**
```
WMA(n) = Σ w[i] × close[i] / Σ w[i],  w[i] = n − i,  i = 0 .. n−1
```

**Parâmetros:** `period: u8` (n)
**Saída:** 1 coluna — `wma`
**Warmup:** n barras

---

### HMA — Hull Moving Average

**Fórmula:**
```
raw = WMA(⌊n/2⌋) × 2 − WMA(n)
HMA = WMA(⌊√n⌋) of raw
```

**Parâmetros:** `period: u8` (n)
**Saída:** 1 coluna — `hma`
**Warmup:** n + ⌊√n⌋ barras

---

### KAMA — Kaufman Adaptive Moving Average

**Fórmula:**
```
ER = |close[t] − close[t−n]| / Σ |close[i] − close[i−1]|,  i = t−n+1 .. t
SC = [ER × (2/(3+1) − 2/(n+1)) + 2/(n+1)]²
KAMA[t] = SC × close[t] + (1 − SC) × KAMA[t−1]
```

**Parâmetros:** `period: u8` (n — período do efficiency ratio)
**Saída:** 1 coluna — `kama`
**Warmup:** n barras

---

### PSAR — Parabolic SAR

**Fórmula:**
```
SAR[t+1] = SAR[t] + AF × (EP − SAR[t])
AF inicia em af_step, incrementa af_step a cada novo EP, até af_max
EP = extreme point (high em uptrend, low em downtrend)
Reversão quando price toca SAR; AF reseta para af_step
```

**Parâmetros:** `af_step: f64` (default 0.02), `af_max: f64` (default 0.2)
**Constraint:** `af_step < af_max`
**Saída:** 2 colunas — `sar`, `trend` (−1, 0, +1)
**Warmup:** 2 barras

---

## Tendência (4)

### ADX — Average Directional Index

**Fórmula:**
```
+DM = max(high[t] − high[t−1], 0)  se > |low[t] − low[t−1]|,  senão 0
−DM = max(low[t−1] − low[t], 0)   se > |high[t] − high[t−1]|,  senão 0
TR  = max(high − low, |high − close[t−1]|, |low − close[t−1]|)
+DI = 100 × RMA(n) de (+DM) / RMA(n) de TR
−DI = 100 × RMA(n) de (−DM) / RMA(n) de TR
DX  = 100 × |+DI − −DI| / (+DI + −DI)
ADX = RMA(n) de DX
ADXR = (ADX + ADX[t−n]) / 2
```

**Parâmetros:** `period: u8` (n) — default 14
**Saída:** 4 colunas — `adx`, `plus_di`, `minus_di`, `adxr`
**Warmup:** ~2n barras

---

### Aroon

**Fórmula:**
```
Aroon Up   = ((n − bars_since_highest_high) / n) × 100
Aroon Down = ((n − bars_since_lowest_low)   / n) × 100
```

**Parâmetros:** `period: u8` (n)
**Saída:** 2 colunas — `up`, `down` (0–100)
**Warmup:** n barras

---

### Ichimoku Cloud

**Fórmula:**
```
Tenkan-sen   = (highest_high(n1) + lowest_low(n1)) / 2
Kijun-sen    = (highest_high(n2) + lowest_low(n2)) / 2
Senkou Span A = (Tenkan + Kijun) / 2,  deslocada n2 barras à frente
Senkou Span B = (highest_high(n3) + lowest_low(n3)) / 2,  deslocada n2 barras à frente
```

**Parâmetros:** `tenkan: u8` (n1, default 9), `kijun: u8` (n2, default 26), `senkou: u8` (n3, default 52)
**Saída:** 4 colunas — `tenkan`, `kijun`, `senkou_a`, `senkou_b`
**Warmup:** n3 barras

---

### Price Channel

**Fórmula:**
```
Upper = highest_high(n)
Lower = lowest_low(n)
```

**Parâmetros:** `period: u8` (n)
**Saída:** 2 colunas — `upper`, `lower`
**Warmup:** n barras

---

## Momentum (14)

### RSI — Relative Strength Index

**Fórmula:**
```
Gain = max(close[t] − close[t−1], 0)
Loss = max(close[t−1] − close[t], 0)
avg_gain = RMA(n) de Gain    (Wilder's smoothing: α = 1/n)
avg_loss = RMA(n) de Loss
RS  = avg_gain / avg_loss
RSI = 100 − 100 / (1 + RS)
```

**Parâmetros:** `period: u8` (n) — default 14
**Saída:** 1 coluna — `rsi` (0–100)
**Warmup:** n barras

---

### MACD — Moving Average Convergence Divergence

**Fórmula:**
```
MACD      = EMA(fast) − EMA(slow)
Signal    = EMA(signal) de MACD
Histogram = MACD − Signal
```

**Parâmetros:** `fast: u8` (default 12), `slow: u8` (default 26), `signal: u8` (default 9)
**Saída:** 3 colunas — `macd`, `signal`, `histogram`
**Warmup:** slow + signal barras

---

### Stochastic Oscillator

**Fórmula:**
```
%K = 100 × (close − lowest_low(n)) / (highest_high(n) − lowest_low(n))
%D = SMA(smooth) de %K
```

**Parâmetros:** `period: u8` (n, default 14), `smooth: u8` (default 3)
**Saída:** 2 colunas — `k`, `d` (0–100)
**Warmup:** n barras

---

### Momentum Index

**Fórmula:**
```
Momentum = close[t] − close[t−slow]
Signal   = close[t] − close[t−fast]
```

> `slow` é o período longo (period1 no yata); `fast` é o período curto (period2).

**Parâmetros:** `slow: u8` (default 10), `fast: u8` (default 1)
**Constraint:** slow > fast
**Saída:** 2 colunas — `momentum`, `signal`
**Warmup:** slow barras

---

### CCI — Commodity Channel Index

**Fórmula:**
```
TP  = (high + low + close) / 3
CCI = (TP − SMA(TP, n)) / (0.015 × mean_deviation(TP, n))

onde mean_deviation = (1/n) × Σ |TP[i] − SMA(TP, n)|
```

**Parâmetros:** `period: u8` (n)
**Saída:** 1 coluna — `cci`
**Warmup:** n barras

---

### CMO — Chande Momentum Oscillator

**Fórmula:**
```
Gains  = Σ max(close[i] − close[i−1], 0),  i = t−n+1 .. t
Losses = Σ max(close[i−1] − close[i], 0),  i = t−n+1 .. t
CMO = 100 × (Gains − Losses) / (Gains + Losses)
```

**Parâmetros:** `period: u8` (n)
**Saída:** 1 coluna — `cmo` (−100 a +100)
**Warmup:** n barras

---

### TRIX

**Fórmula:**
```
TR = EMA(n) de EMA(n) de EMA(n) de close   (triple-smoothed)
TRIX = (TR[t] − TR[t−1]) / TR[t−1]
Signal = EMA(signal) de TRIX
```

**Parâmetros:** `period: u8` (n)
**Saída:** 2 colunas — `trix`, `signal`
**Warmup:** ~3n barras

---

### TSI — True Strength Index

**Fórmula:**
```
m = close[t] − close[t−1]
ds = EMA(slow) de EMA(fast) de m        (double-smoothed momentum)
dl = EMA(slow) de EMA(fast) de |m|       (double-smoothed abs momentum)
TSI = 100 × ds / dl
Signal = EMA(signal) de TSI
```

**Parâmetros:** `slow: u8` (period1, default 25), `fast: u8` (period2, default 13), `signal: u8` (period3, default 13)
**Constraint:** fast ≤ slow
**Saída:** 2 colunas — `tsi`, `signal`
**Warmup:** slow + signal barras

---

### Coppock Curve

**Fórmula:**
```
ROC_long  = ROC(slow) de close   = 100 × (close[t] − close[t−slow]) / close[t−slow]
ROC_short = ROC(fast) de close   = 100 × (close[t] − close[t−fast]) / close[t−fast]
Coppock = WMA(signal) de (ROC_long + ROC_short)
Signal  = EMA(signal) de Coppock
```

**Parâmetros:** `fast: u8` (default 11), `slow: u8` (default 14), `signal: u8` (default 10)
**Saída:** 2 colunas — `coppock`, `signal`
**Warmup:** slow + signal barras

---

### DPO — Detrended Price Oscillator

**Fórmula:**
```
DPO = close[t] − SMA(n)[t − ⌊n/2⌋ + 1]
```

> Remove a tendência centrando o preço em relação à SMA deslocada.

**Parâmetros:** `period: u8` (n) — default 20
**Saída:** 1 coluna — `dpo`
**Warmup:** n barras

---

### Fisher Transform

**Fórmula:**
```
value  = 2 × (close − lowest_low(n)) / (highest_high(n) − lowest_low(n)) − 1
Fisher = 0.5 × ln((1 + value) / (1 − value)) + 0.5 × Fisher[t−1]
Signal = Fisher[t−1]
```

> Transforma uma distribuição arbitrária em algo próximo da normal (gaussiana).

**Parâmetros:** `period: u8` (n)
**Saída:** 2 colunas — `fisher`, `signal`
**Warmup:** n barras

---

### Awesome Oscillator

**Fórmula:**
```
median_price = (high + low) / 2
AO = SMA(fast)(median_price) − SMA(slow)(median_price)
```

> Nota: no yata, `ma1` = slow, `ma2` = fast.

**Parâmetros:** `fast: u8` (default 5), `slow: u8` (default 34)
**Saída:** 1 coluna — `ao`
**Warmup:** slow barras

---

### SMI Ergodic Indicator

**Fórmula:**
```
m = close[t] − close[t−1]
ds = EMA(slow) de EMA(fast) de m
dl = EMA(slow) de EMA(fast) de |m|
SMI = 100 × ds / dl
Signal = EMA(signal) de SMI
Histogram = SMI − Signal
```

> O ct-mcp-server escala a saída por ×100 (yata retorna em [-1,1]/[-2,2]).

**Parâmetros:** `slow: u8` (period1, default 20), `fast: u8` (period2, default 5), `signal: u8` (default 5)
**Constraint:** fast ≤ slow
**Saída:** 3 colunas — `smi`, `signal`, `histogram` (escala ×100)
**Warmup:** slow + signal barras

---

### RVI — Relative Vigor Index

**Fórmula:**
```
NUM = SMA(n) de (close − open)
DEN = SMA(n) de (high − low)
RVI = NUM / DEN
Signal = média simétrica de 4 períodos de RVI
```

**Parâmetros:** `period: u8` (n)
**Saída:** 2 colunas — `rvi`, `signal`
**Warmup:** n barras

---

### KST — Know Sure Thing

**Fórmula:**
```
ROC1 = SMA(signal) de ROC(period1)
ROC2 = SMA(signal) de ROC(period2)
ROC3 = SMA(signal) de ROC(period3)
ROC4 = SMA(signal) de ROC(period4)
KST = ROC1×1 + ROC2×2 + ROC3×3 + ROC4×4
Signal = SMA(signal) de KST
```

> Mapeamento: period1=fast, period2=fast+5, period3=slow, period4=slow+10

**Parâmetros:** `fast: u8` (default 10), `slow: u8` (default 20), `signal: u8` (default 10)
**Constraint:** fast < slow (garante period1 < period2 < period3 < period4)
**Saída:** 2 colunas — `kst`, `signal`
**Warmup:** slow+10 + signal barras

---

### Woodies CCI

**Fórmula:**
```
TP = (high + low + close) / 3
Turbo = (TP − SMA(TP, fast)) / (0.015 × mean_deviation(TP, fast))
Trend = (TP − SMA(TP, slow)) / (0.015 × mean_deviation(TP, slow))
```

> Dois CCIs de períodos diferentes (turbo curto, trend longo).

**Parâmetros:** `fast: u8` (period1, default 6), `slow: u8` (period2, default 14)
**Constraint:** fast < slow
**Saída:** 2 colunas — `turbo`, `trend`
**Warmup:** slow barras

---

## Volatilidade (6)

### ATR — Average True Range

**Fórmula:**
```
TR  = max(high − low, |high − close[t−1]|, |low − close[t−1]|)
ATR = RMA(n) de TR    (Wilder's smoothing: α = 1/n)
```

**Parâmetros:** `period: u8` (n) — default 14
**Saída:** 1 coluna — `atr`
**Warmup:** n barras

---

### Bollinger Bands

**Fórmula:**
```
Middle = SMA(n) de close
σ      = desvio padrão de close over n barras
Upper  = Middle + k × σ
Lower  = Middle − k × σ
onde k = 2 (fixo no ct-mcp-server)
```

**Parâmetros:** `period: u8` (n) — default 20
**Saída:** 3 colunas — `upper`, `middle`, `lower`
**Warmup:** n barras

---

### Donchian Channel

**Fórmula:**
```
Upper  = highest_high(n)
Lower  = lowest_low(n)
Middle = (Upper + Lower) / 2
```

**Parâmetros:** `period: u8` (n)
**Saída:** 3 colunas — `lower`, `middle`, `upper`
**Warmup:** n barras

---

### Keltner Channel

**Fórmula:**
```
Middle = EMA(n) de close
ATR    = RMA(n) de TR
Upper  = Middle + 2 × ATR
Lower  = Middle − 2 × ATR
```

**Parâmetros:** `period: u8` (n) — default 20
**Saída:** 3 colunas — `middle`, `upper`, `lower`
**Warmup:** n barras

---

### Envelopes

**Fórmula:**
```
Upper  = SMA(n) × (1 + k)
Lower  = SMA(n) × (1 − k)
Source = SMA(n)
onde k = 0.1 (10%, fixo)
```

**Parâmetros:** `period: u8` (n) — default 20
**Saída:** 3 colunas — `upper`, `lower`, `source`
**Warmup:** n barras

---

### Chande Kroll Stop

**Fórmula:**
```
ATR = RMA(n) de TR
Stop Long  = highest_high(n) − q × ATR
Stop Short = lowest_low(n)   + q × ATR
Source     = close
```

**Parâmetros:** `period: u8` (n, default 10), `q: u8` (default 9)
**Saída:** 3 colunas — `stop_long`, `source`, `stop_short`
**Warmup:** n barras

---

## Volume (5)

### OBV — On-Balance Volume

**Fórmula:**
```
if close > close[t−1]:  OBV[t] = OBV[t−1] + volume
if close < close[t−1]:  OBV[t] = OBV[t−1] − volume
if close = close[t−1]:  OBV[t] = OBV[t−1]
```

**Parâmetros:** nenhum
**Saída:** 1 coluna — `obv`
**Warmup:** 1 barra

---

### CMF — Chaikin Money Flow

**Fórmula:**
```
MFV = ((close − low) − (high − close)) / (high − low) × volume
CMF = (Σ MFV) / (Σ volume)  over n barras
```

> Se high = low (doji), MFV = 0.

**Parâmetros:** `period: u8` (n) — default 20
**Saída:** 1 coluna — `cmf` (−1 a +1)
**Warmup:** n barras

---

### MFI — Money Flow Index

**Fórmula:**
```
TP = (high + low + close) / 3
MF = TP × volume
positive_mf = Σ MF  quando TP > TP[t−1],  over n barras
negative_mf = Σ MF  quando TP < TP[t−1],  over n barras
MR = positive_mf / negative_mf
MFI = 100 − 100 / (1 + MR)
```

**Parâmetros:** `period: u8` (n) — default 14
**Saída:** 3 colunas — `upper` (zona overbought), `mfi` (0–100), `lower` (zona oversold)
**Warmup:** n barras

---

### EFI — Elder's Force Index

**Fórmula:**
```
FI  = (close − close[t−1]) × volume
EFI = EMA(n) de FI
```

**Parâmetros:** `period: u8` (n) — default 13
**Saída:** 1 coluna — `efi`
**Warmup:** n barras

---

### EOM — Ease of Movement

**Fórmula:**
```
Distance  = ((high + low) / 2) − ((high[t−1] + low[t−1]) / 2)
BoxRatio  = volume / (high − low)
EMV       = Distance / BoxRatio
EOM       = SMA(n) de EMV
```

> Se high = low, EMV = 0.

**Parâmetros:** `period: u8` (n) — default 14
**Saída:** 1 coluna — `eom`
**Warmup:** n barras

---

> Próximo: [Pipeline declarativo](./03-pipeline-declarativo.pt.md) · [Indicadores premium](./02-indicadores-premium.pt.md)
