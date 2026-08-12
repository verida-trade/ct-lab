# Technical Reference — Formulas for the 36 Public Indicators

> Mathematical definition of each indicator. No signal interpretation — just the formulation, parameters, and outputs.

---

## Moving Averages (6)

### SMA — Simple Moving Average

**Formula:**
```
SMA(n) = (1/n) × Σ close[i],  i = t−n+1 .. t
```

**Parameters:** `period: u8` (n) — default 20
**Output:** 1 column — `sma`
**Warmup:** n bars

---

### EMA — Exponential Moving Average

**Formula:**
```
α = 2 / (n + 1)
EMA[t] = α × close[t] + (1 − α) × EMA[t−1]
EMA[0] = close[0]
```

**Parameters:** `period: u8` (n) — default 20
**Output:** 1 column — `ema`
**Warmup:** ~2n bars (convergence)

---

### WMA — Weighted Moving Average

**Formula:**
```
WMA(n) = Σ w[i] × close[i] / Σ w[i],  w[i] = n − i,  i = 0 .. n−1
```

**Parameters:** `period: u8` (n)
**Output:** 1 column — `wma`
**Warmup:** n bars

---

### HMA — Hull Moving Average

**Formula:**
```
raw = WMA(⌊n/2⌋) × 2 − WMA(n)
HMA = WMA(⌊√n⌋) of raw
```

**Parameters:** `period: u8` (n)
**Output:** 1 column — `hma`
**Warmup:** n + ⌊√n⌋ bars

---

### KAMA — Kaufman Adaptive Moving Average

**Formula:**
```
ER = |close[t] − close[t−n]| / Σ |close[i] − close[i−1]|,  i = t−n+1 .. t
SC = [ER × (2/(3+1) − 2/(n+1)) + 2/(n+1)]²
KAMA[t] = SC × close[t] + (1 − SC) × KAMA[t−1]
```

**Parameters:** `period: u8` (n — efficiency ratio period)
**Output:** 1 column — `kama`
**Warmup:** n bars

---

### PSAR — Parabolic SAR

**Formula:**
```
SAR[t+1] = SAR[t] + AF × (EP − SAR[t])
AF starts at af_step, increments by af_step on each new EP, up to af_max
EP = extreme point (high in uptrend, low in downtrend)
Reversal when price touches SAR; AF resets to af_step
```

**Parameters:** `af_step: f64` (default 0.02), `af_max: f64` (default 0.2)
**Constraint:** `af_step < af_max`
**Output:** 2 columns — `sar`, `trend` (−1, 0, +1)
**Warmup:** 2 bars

---

## Trend (4)

### ADX — Average Directional Index

**Formula:**
```
+DM = max(high[t] − high[t−1], 0)  if > |low[t] − low[t−1]|,  else 0
−DM = max(low[t−1] − low[t], 0)   if > |high[t] − high[t−1]|,  else 0
TR  = max(high − low, |high − close[t−1]|, |low − close[t−1]|)
+DI = 100 × RMA(n) of (+DM) / RMA(n) of TR
−DI = 100 × RMA(n) of (−DM) / RMA(n) of TR
DX  = 100 × |+DI − −DI| / (+DI + −DI)
ADX = RMA(n) of DX
ADXR = (ADX + ADX[t−n]) / 2
```

**Parameters:** `period: u8` (n) — default 14
**Output:** 4 columns — `adx`, `plus_di`, `minus_di`, `adxr`
**Warmup:** ~2n bars

---

### Aroon

**Formula:**
```
Aroon Up   = ((n − bars_since_highest_high) / n) × 100
Aroon Down = ((n − bars_since_lowest_low)   / n) × 100
```

**Parameters:** `period: u8` (n)
**Output:** 2 columns — `up`, `down` (0–100)
**Warmup:** n bars

---

### Ichimoku Cloud

**Formula:**
```
Tenkan-sen    = (highest_high(n1) + lowest_low(n1)) / 2
Kijun-sen     = (highest_high(n2) + lowest_low(n2)) / 2
Senkou Span A = (Tenkan + Kijun) / 2,  shifted forward n2 bars
Senkou Span B = (highest_high(n3) + lowest_low(n3)) / 2,  shifted forward n2 bars
```

**Parameters:** `tenkan: u8` (n1, default 9), `kijun: u8` (n2, default 26), `senkou: u8` (n3, default 52)
**Output:** 4 columns — `tenkan`, `kijun`, `senkou_a`, `senkou_b`
**Warmup:** n3 bars

---

### Price Channel

**Formula:**
```
Upper = highest_high(n)
Lower = lowest_low(n)
```

**Parameters:** `period: u8` (n)
**Output:** 2 columns — `upper`, `lower`
**Warmup:** n bars

---

## Momentum (14)

### RSI — Relative Strength Index

**Formula:**
```
Gain = max(close[t] − close[t−1], 0)
Loss = max(close[t−1] − close[t], 0)
avg_gain = RMA(n) of Gain    (Wilder's smoothing: α = 1/n)
avg_loss = RMA(n) of Loss
RS  = avg_gain / avg_loss
RSI = 100 − 100 / (1 + RS)
```

**Parameters:** `period: u8` (n) — default 14
**Output:** 1 column — `rsi` (0–100)
**Warmup:** n bars

---

### MACD — Moving Average Convergence Divergence

**Formula:**
```
MACD      = EMA(fast) − EMA(slow)
Signal    = EMA(signal) of MACD
Histogram = MACD − Signal
```

**Parameters:** `fast: u8` (default 12), `slow: u8` (default 26), `signal: u8` (default 9)
**Output:** 3 columns — `macd`, `signal`, `histogram`
**Warmup:** slow + signal bars

---

### Stochastic Oscillator

**Formula:**
```
%K = 100 × (close − lowest_low(n)) / (highest_high(n) − lowest_low(n))
%D = SMA(smooth) of %K
```

**Parameters:** `period: u8` (n, default 14), `smooth: u8` (default 3)
**Output:** 2 columns — `k`, `d` (0–100)
**Warmup:** n bars

---

### Momentum Index

**Formula:**
```
Momentum = close[t] − close[t−slow]
Signal   = close[t] − close[t−fast]
```

> `slow` is the long period (period1 in yata); `fast` is the short period (period2).

**Parameters:** `slow: u8` (default 10), `fast: u8` (default 1)
**Constraint:** slow > fast
**Output:** 2 columns — `momentum`, `signal`
**Warmup:** slow bars

---

### CCI — Commodity Channel Index

**Formula:**
```
TP  = (high + low + close) / 3
CCI = (TP − SMA(TP, n)) / (0.015 × mean_deviation(TP, n))

where mean_deviation = (1/n) × Σ |TP[i] − SMA(TP, n)|
```

**Parameters:** `period: u8` (n)
**Output:** 1 column — `cci`
**Warmup:** n bars

---

### CMO — Chande Momentum Oscillator

**Formula:**
```
Gains  = Σ max(close[i] − close[i−1], 0),  i = t−n+1 .. t
Losses = Σ max(close[i−1] − close[i], 0),  i = t−n+1 .. t
CMO = 100 × (Gains − Losses) / (Gains + Losses)
```

**Parameters:** `period: u8` (n)
**Output:** 1 column — `cmo` (−100 to +100)
**Warmup:** n bars

---

### TRIX

**Formula:**
```
TR = EMA(n) of EMA(n) of EMA(n) of close   (triple-smoothed)
TRIX = (TR[t] − TR[t−1]) / TR[t−1]
Signal = EMA(signal) of TRIX
```

**Parameters:** `period: u8` (n)
**Output:** 2 columns — `trix`, `signal`
**Warmup:** ~3n bars

---

### TSI — True Strength Index

**Formula:**
```
m = close[t] − close[t−1]
ds = EMA(slow) of EMA(fast) of m        (double-smoothed momentum)
dl = EMA(slow) of EMA(fast) of |m|       (double-smoothed abs momentum)
TSI = 100 × ds / dl
Signal = EMA(signal) of TSI
```

**Parameters:** `slow: u8` (period1, default 25), `fast: u8` (period2, default 13), `signal: u8` (period3, default 13)
**Constraint:** fast ≤ slow
**Output:** 2 columns — `tsi`, `signal`
**Warmup:** slow + signal bars

---

### Coppock Curve

**Formula:**
```
ROC_long  = ROC(slow) of close   = 100 × (close[t] − close[t−slow]) / close[t−slow]
ROC_short = ROC(fast) of close   = 100 × (close[t] − close[t−fast]) / close[t−fast]
Coppock = WMA(signal) of (ROC_long + ROC_short)
Signal  = EMA(signal) of Coppock
```

**Parameters:** `fast: u8` (default 11), `slow: u8` (default 14), `signal: u8` (default 10)
**Output:** 2 columns — `coppock`, `signal`
**Warmup:** slow + signal bars

---

### DPO — Detrended Price Oscillator

**Formula:**
```
DPO = close[t] − SMA(n)[t − ⌊n/2⌋ + 1]
```

> Removes trend by centering price against a displaced SMA.

**Parameters:** `period: u8` (n) — default 20
**Output:** 1 column — `dpo`
**Warmup:** n bars

---

### Fisher Transform

**Formula:**
```
value  = 2 × (close − lowest_low(n)) / (highest_high(n) − lowest_low(n)) − 1
Fisher = 0.5 × ln((1 + value) / (1 − value)) + 0.5 × Fisher[t−1]
Signal = Fisher[t−1]
```

> Transforms an arbitrary distribution into something close to a normal (Gaussian) distribution.

**Parameters:** `period: u8` (n)
**Output:** 2 columns — `fisher`, `signal`
**Warmup:** n bars

---

### Awesome Oscillator

**Formula:**
```
median_price = (high + low) / 2
AO = SMA(fast)(median_price) − SMA(slow)(median_price)
```

> Note: in yata, `ma1` = slow, `ma2` = fast.

**Parameters:** `fast: u8` (default 5), `slow: u8` (default 34)
**Output:** 1 column — `ao`
**Warmup:** slow bars

---

### SMI Ergodic Indicator

**Formula:**
```
m = close[t] − close[t−1]
ds = EMA(slow) of EMA(fast) of m
dl = EMA(slow) of EMA(fast) of |m|
SMI = 100 × ds / dl
Signal = EMA(signal) of SMI
Histogram = SMI − Signal
```

> The ct-mcp-server scales the output by ×100 (yata returns values in [-1,1]/[-2,2]).

**Parameters:** `slow: u8` (period1, default 20), `fast: u8` (period2, default 5), `signal: u8` (default 5)
**Constraint:** fast ≤ slow
**Output:** 3 columns — `smi`, `signal`, `histogram` (scaled ×100)
**Warmup:** slow + signal bars

---

### RVI — Relative Vigor Index

**Formula:**
```
NUM = SMA(n) of (close − open)
DEN = SMA(n) of (high − low)
RVI = NUM / DEN
Signal = symmetric 4-period average of RVI
```

**Parameters:** `period: u8` (n)
**Output:** 2 columns — `rvi`, `signal`
**Warmup:** n bars

---

### KST — Know Sure Thing

**Formula:**
```
ROC1 = SMA(signal) of ROC(period1)
ROC2 = SMA(signal) of ROC(period2)
ROC3 = SMA(signal) of ROC(period3)
ROC4 = SMA(signal) of ROC(period4)
KST = ROC1×1 + ROC2×2 + ROC3×3 + ROC4×4
Signal = SMA(signal) of KST
```

> Mapping: period1=fast, period2=fast+5, period3=slow, period4=slow+10

**Parameters:** `fast: u8` (default 10), `slow: u8` (default 20), `signal: u8` (default 10)
**Constraint:** fast < slow (ensures period1 < period2 < period3 < period4)
**Output:** 2 columns — `kst`, `signal`
**Warmup:** slow+10 + signal bars

---

### Woodies CCI

**Formula:**
```
TP = (high + low + close) / 3
Turbo = (TP − SMA(TP, fast)) / (0.015 × mean_deviation(TP, fast))
Trend = (TP − SMA(TP, slow)) / (0.015 × mean_deviation(TP, slow))
```

> Two CCIs of different periods (short turbo, long trend).

**Parameters:** `fast: u8` (period1, default 6), `slow: u8` (period2, default 14)
**Constraint:** fast < slow
**Output:** 2 columns — `turbo`, `trend`
**Warmup:** slow bars

---

## Volatility (6)

### ATR — Average True Range

**Formula:**
```
TR  = max(high − low, |high − close[t−1]|, |low − close[t−1]|)
ATR = RMA(n) of TR    (Wilder's smoothing: α = 1/n)
```

**Parameters:** `period: u8` (n) — default 14
**Output:** 1 column — `atr`
**Warmup:** n bars

---

### Bollinger Bands

**Formula:**
```
Middle = SMA(n) of close
σ      = standard deviation of close over n bars
Upper  = Middle + k × σ
Lower  = Middle − k × σ
where k = 2 (fixed in ct-mcp-server)
```

**Parameters:** `period: u8` (n) — default 20
**Output:** 3 columns — `upper`, `middle`, `lower`
**Warmup:** n bars

---

### Donchian Channel

**Formula:**
```
Upper  = highest_high(n)
Lower  = lowest_low(n)
Middle = (Upper + Lower) / 2
```

**Parameters:** `period: u8` (n)
**Output:** 3 columns — `lower`, `middle`, `upper`
**Warmup:** n bars

---

### Keltner Channel

**Formula:**
```
Middle = EMA(n) of close
ATR    = RMA(n) of TR
Upper  = Middle + 2 × ATR
Lower  = Middle − 2 × ATR
```

**Parameters:** `period: u8` (n) — default 20
**Output:** 3 columns — `middle`, `upper`, `lower`
**Warmup:** n bars

---

### Envelopes

**Formula:**
```
Upper  = SMA(n) × (1 + k)
Lower  = SMA(n) × (1 − k)
Source = SMA(n)
where k = 0.1 (10%, fixed)
```

**Parameters:** `period: u8` (n) — default 20
**Output:** 3 columns — `upper`, `lower`, `source`
**Warmup:** n bars

---

### Chande Kroll Stop

**Formula:**
```
ATR = RMA(n) of TR
Stop Long  = highest_high(n) − q × ATR
Stop Short = lowest_low(n)   + q × ATR
Source     = close
```

**Parameters:** `period: u8` (n, default 10), `q: u8` (default 9)
**Output:** 3 columns — `stop_long`, `source`, `stop_short`
**Warmup:** n bars

---

## Volume (5)

### OBV — On-Balance Volume

**Formula:**
```
if close > close[t−1]:  OBV[t] = OBV[t−1] + volume
if close < close[t−1]:  OBV[t] = OBV[t−1] − volume
if close = close[t−1]:  OBV[t] = OBV[t−1]
```

**Parameters:** none
**Output:** 1 column — `obv`
**Warmup:** 1 bar

---

### CMF — Chaikin Money Flow

**Formula:**
```
MFV = ((close − low) − (high − close)) / (high − low) × volume
CMF = (Σ MFV) / (Σ volume)  over n bars
```

> If high = low (doji), MFV = 0.

**Parameters:** `period: u8` (n) — default 20
**Output:** 1 column — `cmf` (−1 to +1)
**Warmup:** n bars

---

### MFI — Money Flow Index

**Formula:**
```
TP = (high + low + close) / 3
MF = TP × volume
positive_mf = Σ MF  when TP > TP[t−1],  over n bars
negative_mf = Σ MF  when TP < TP[t−1],  over n bars
MR = positive_mf / negative_mf
MFI = 100 − 100 / (1 + MR)
```

**Parameters:** `period: u8` (n) — default 14
**Output:** 3 columns — `upper` (overbought zone), `mfi` (0–100), `lower` (oversold zone)
**Warmup:** n bars

---

### EFI — Elder's Force Index

**Formula:**
```
FI  = (close − close[t−1]) × volume
EFI = EMA(n) of FI
```

**Parameters:** `period: u8` (n) — default 13
**Output:** 1 column — `efi`
**Warmup:** n bars

---

### EOM — Ease of Movement

**Formula:**
```
Distance  = ((high + low) / 2) − ((high[t−1] + low[t−1]) / 2)
BoxRatio  = volume / (high − low)
EMV       = Distance / BoxRatio
EOM       = SMA(n) of EMV
```

> If high = low, EMV = 0.

**Parameters:** `period: u8` (n) — default 14
**Output:** 1 column — `eom`
**Warmup:** n bars

---

> Next: [Pipeline declarativo](./03-pipeline-declarativo.en.md) · [Indicadores premium](./02-indicadores-premium.en.md)
