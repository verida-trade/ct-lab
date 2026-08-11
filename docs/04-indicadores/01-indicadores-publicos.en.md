# Built-in Indicators (36 Public)

> Full catalog of technical indicators available without a premium license.

The `ct-mcp-server` exposes **36 public indicators** based on the [yata](https://crates.io/crates/yata) `0.7.0` library, plus 2 customs (`atr`, `obv`). All follow the same call and return pattern.

---

## Call pattern

Every indicator tool receives a series URI and its own parameters, and returns **URI + meta** (never raw rows):

```json
{
  "name": "rsi",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "period": 14
  }
}
```

**Return:**
```json
{
  "uri": "ct://derived/rsi_close_14_680c3e",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "valid_from_ts": 1784068200,
  "value_names": ["rsi"],
  "latest": [54.32],
  "evicted_series": [],
  "plot": null
}
```

| Field | Meaning |
|---|---|
| `uri` | Persisted URI of the derived series (`ct://derived/<name>`) |
| `valid_from_ts` | Where the indicator stops being NaN (warmup awareness) |
| `value_names` | Names of produced columns |
| `latest` | Latest computed values (aligned with `value_names`) |
| `evicted_series` | Series removed from cache (LRU) to make room |

> **Tip:** the derived name is auto-generated (`<ind>_<col>_<params>_<hash6>`) if you don't pass `name`. Pass `name` for a semantic name (`"btc_rsi14"`).

---

## Catalog by category

### Trend (moving averages)

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `sma` | Simple Moving Average | `period` | `sma` |
| `ema` | Exponential Moving Average | `period` | `ema` |
| `wma` | Weighted Moving Average | `period` | `wma` |
| `hma` | Hull Moving Average | `period` | `hma` |
| `kama` | Kaufman Adaptive MA | `period` | `kama` |
| `psar` | Parabolic SAR | (default) | `psar` |
| `aroon` | Aroon Up/Down | `period` | `aroon_up`, `aroon_down` |
| `ichimoku` | Ichimoku Cloud | (default 9,26,52) | `tenkan`, `kijun`, `senkou_a`, `senkou_b`, `chikou` |

### Momentum

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `rsi` | Relative Strength Index | `period` | `rsi` |
| `macd` | MACD | (fixed 12,26,9) | `macd`, `signal`, `hist` |
| `stochastic` | Stochastic Oscillator | (fixed 14,3) | `k`, `d` |
| `momentum_idx` | Momentum Index | (default 10,1) | `momentum` |
| `cci` | Commodity Channel Index | `period` | `cci` |
| `cmo` | Chande Momentum Oscillator | `period` | `cmo` |
| `trix` | TRIX | `period` | `trix` |
| `tsi` | True Strength Index | (default 25,13,13) | `tsi` |
| `coppock` | Coppock Curve | (default 14,11,10) | `coppock` |
| `dpo` | Detrended Price Oscillator | (default 21) | `dpo` |
| `fisher` | Fisher Transform | `period` | `fisher` |
| `awesome` | Awesome Oscillator | (default 5,34) | `awesome` |
| `smi` | SMI Ergodic Indicator | (default 20,5,5) | `smi` |
| `rvi` | Relative Vigor Index | `period` | `rvi` |
| `kst` | Know Sure Thing | (default) | `kst`, `signal` |
| `woodies` | Woodies CCI | (default 6,14) | `cci`, `signal` |

### Volatility

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `atr` | Average True Range | `period` | `atr` |
| `bollinger` | Bollinger Bands | (fixed 20, 2σ) | `upper`, `middle`, `lower` |
| `donchian` | Donchian Channel | `period` | `upper`, `lower` |
| `keltner` | Keltner Channel | (default) | `upper`, `middle`, `lower` |
| `envelopes` | Moving Envelopes | (default SMA20, k=0.1) | `upper`, `lower` |
| `chande_kroll` | Chande Kroll Stop | (default) | `stop_long`, `stop_short` |
| `price_channel` | Price Channel | `period` | `upper`, `lower` |

### Volume

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `obv` | On-Balance Volume | — | `obv` |
| `cmf` | Chaikin Money Flow | `period` | `cmf` |
| `mfi` | Money Flow Index | `period` | `mfi` |
| `efi` | Elder's Force Index | (default) | `efi` |
| `eom` | Ease of Movement | (default) | `eom` |
| `klinger` | Klinger Volume Oscillator | (default 34,55) | `kvo`, `signal` |
| `chaikin_osc` | Chaikin Oscillator | (default 3,10) | `chaikin_osc` |

### Trend Strength

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `adx` | Average Directional Index | (default) | `adx`, `di_plus`, `di_minus` |
| `trend_strength` | Trend Strength Index | `period` | `tsi` |

---

## Smart column defaults

For single-column indicators (`sma`, `ema`, `rsi`, etc.), if you omit `column`, the system uses:
- `"close"` if it exists in the series
- The only available column, if there's just one
- Error if there are multiple columns and none is `"close"`

For multi-column indicators (`atr`, `bollinger`, `ichimoku`, etc.), defaults are `"high"`, `"low"`, `"close"`, `"volume"` as appropriate.

---

## Practical examples

### RSI(14) on BTCUSDT 15m

**Chat:**
> Calculate RSI with 14 periods on BTCUSDT 15m

```json
{ "name": "rsi", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 14 } }
```

### MACD (12,26,9) with custom name

**Chat:**
> Calculate MACD on BTCUSDT 15m and name it "btc_macd"

```json
{ "name": "macd", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "name": "btc_macd" } }
```

**Return:**
```json
{
  "uri": "ct://derived/btc_macd",
  "value_names": ["macd", "signal", "hist"],
  "latest": [12.34, 10.5, 1.84]
}
```

### Bollinger Bands (3 columns)

```json
{ "name": "bollinger", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m" } }
```

**Return:**
```json
{
  "uri": "ct://derived/bollinger_close_20_2sigma_abc123",
  "value_names": ["upper", "middle", "lower"],
  "latest": [64200.0, 63700.0, 63200.0]
}
```

---

## Chain walking (indicator on indicator)

All indicator tools accept a **derived** URI as input (not just raw). The system follows the `source_uris` chain recursively to find the original raw:

```json
{ "name": "ema", "arguments": { "uri": "ct://derived/btc_macd", "period": 9 } }
```

This computes EMA(9) on the `"close"` column of the MACD derived. To choose another column, pass `column`:

```json
{ "name": "ema", "arguments": { "uri": "ct://derived/btc_macd", "period": 9, "column": "hist" } }
```

---

## Live catalog

The list above may go stale. To always see the current version:

**Chat:**
> List all available indicators

The model reads the `ct://indicators/catalog` resource and returns the complete list with parameters and output columns.

---

> **Premium CT indicators?** See [02-indicadores-premium](./02-indicadores-premium.en.md).
> **Combine indicators?** See [Declarative pipeline](./03-pipeline-declarativo.en.md) and [Vectorized Rhai](./05-rhai-vetorizado.en.md).
