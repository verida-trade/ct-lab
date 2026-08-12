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
| `psar` | Parabolic SAR | `af_step`, `af_max` | `sar`, `trend` |
| `aroon` | Aroon Up/Down | `period` | `up`, `down` |
| `ichimoku` | Ichimoku Cloud | `tenkan`, `kijun`, `senkou` | `tenkan`, `kijun`, `senkou_a`, `senkou_b` |

### Momentum

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `rsi` | Relative Strength Index | `period` | `rsi` |
| `macd` | MACD | `fast`, `slow`, `signal` | `macd`, `signal` |
| `stochastic` | Stochastic Oscillator | `period`, `smooth` | `k`, `d` |
| `momentum_idx` | Momentum Index | `slow`, `fast` | `momentum`, `signal` |
| `cci` | Commodity Channel Index | `period` | `cci` |
| `cmo` | Chande Momentum Oscillator | `period` | `cmo` |
| `trix` | TRIX | `period` | `trix`, `signal` |
| `tsi` | True Strength Index | `slow`, `fast`, `signal` | `tsi`, `signal` |
| `coppock` | Coppock Curve | `fast`, `slow`, `signal` | `coppock`, `signal` |
| `dpo` | Detrended Price Oscillator | `period` | `dpo` |
| `fisher` | Fisher Transform | `period` | `fisher`, `signal` |
| `awesome` | Awesome Oscillator | `fast`, `slow` | `ao` |
| `smi` | SMI Ergodic Indicator | `slow`, `fast`, `signal` | `smi`, `signal`, `histogram` |
| `rvi` | Relative Vigor Index | `period` | `rvi`, `signal` |
| `kst` | Know Sure Thing | `fast`, `slow`, `signal` | `kst`, `signal` |
| `woodies` | Woodies CCI | `fast`, `slow` | `turbo`, `trend` |

### Volatility

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `atr` | Average True Range | `period` | `atr` |
| `bollinger` | Bollinger Bands | `period` | `upper`, `middle`, `lower` |
| `donchian` | Donchian Channel | `period` | `lower`, `middle`, `upper` |
| `keltner` | Keltner Channel | `period` | `middle`, `upper`, `lower` |
| `envelopes` | Moving Envelopes | `period` | `upper`, `lower`, `source` |
| `chande_kroll` | Chande Kroll Stop | `period`, `q` | `stop_long`, `source`, `stop_short` |
| `price_channel` | Price Channel | `period` | `upper`, `lower` |

### Volume

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `obv` | On-Balance Volume | — | `obv` |
| `cmf` | Chaikin Money Flow | `period` | `cmf` |
| `mfi` | Money Flow Index | `period` | `upper`, `mfi`, `lower` |
| `efi` | Elder's Force Index | `period` | `efi` |
| `eom` | Ease of Movement | `period` | `eom` |
| `klinger` | Klinger Volume Oscillator | `fast`, `slow`, `signal` | `klinger`, `signal` |
| `chaikin_osc` | Chaikin Oscillator | `fast`, `slow` | `chaikin_osc` |

### Trend Strength

| Tool | Indicator | Parameters | Output columns |
|---|---|---|---|
| `adx` | Average Directional Index | `period` | `adx`, `plus_di`, `minus_di` |
| `trend_strength` | Trend Strength Index | `period` | `trend_strength` |

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
{ "name": "macd", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "name": "btc_macd", "fast": 12, "slow": 26, "signal": 9 } }
```

**Return:**
```json
{
  "uri": "ct://derived/btc_macd",
  "value_names": ["macd", "signal"],
  "latest": [12.34, 10.5, 1.84]
}
```

### Bollinger Bands (3 columns)

```json
{ "name": "bollinger", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 20 } }
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
{ "name": "ema", "arguments": { "uri": "ct://derived/btc_macd", "period": 9, "column": "signal" } }
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
