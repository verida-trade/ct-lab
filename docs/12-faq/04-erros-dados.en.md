# Data Errors

> Common issues with series, URIs, cache and ingestion.

## "Series not found" / URI doesn't resolve

```
Error: series not found: ct://series/binance/BTCUSDT/15m
```

**Cause:** The series was never fetched or was evicted from cache (LRU eviction in Premium).

**Solution:**
1. Check if the series is in cache:
```python
listar_series()  # lists all series in cache
```
2. If not, fetch it again:
```python
buscar_serie(provider="binance", symbol="BTCUSDT", timeframe="15m")
```

## Series limit reached (1/1)

```
Error: series limit reached (1/1)
```

**Cause:** The Free plan allows only 1 series in cache.

**Solution:**
```python
# Remove the current series before fetching another
remover_serie(uri="ct://series/binance/BTCUSDT/15m")
buscar_serie(provider="binance", symbol="ETHUSDT", timeframe="15m")
```

Or upgrade to Premium (100 simultaneous series with automatic LRU).

## "Column not found: close"

```
Error: column not found: close
```

**Cause:** The series doesn't have the expected column. It may be a synthetic series (without OHLCV) or the column name is different.

**Solution:**
1. List available columns:
```python
info = info_serie(uri="ct://derived/my_indicator")
# → shows value_names and available columns
```
2. Use the correct column name instead of `"close"`

## Backfill error: "bulk dump not available"

**Cause:** Binance doesn't provide bulk dumps for all timeframes or the requested period is too old.

**Solution:**
1. Use `buscar_serie_historico` (downloads in chunks instead of bulk)
2. Reduce the period (e.g.: 30 days instead of 1 year)
3. Use a larger timeframe (1h or 1d instead of 1m) — bulk dumps are more available

## CSV imports but data is wrong

**Cause:** CSV format doesn't match expectations (columns, separator, timestamp).

**Solution:**
1. CSV must have columns: `timestamp,open,high,low,close,volume`
2. Timestamp in **seconds** (Unix epoch), not milliseconds
3. Separator: comma (`,`)

```csv
timestamp,open,high,low,close,volume
1700000000,67100.5,67200.0,67050.0,67150.2,1234.5
1700003600,67150.2,67250.0,67100.0,67200.8,2345.6
```

## Indicator returns NaN for the first N bars

**Cause:** Indicators need a minimum period ("warmup") before producing valid values. For example, SMA(20) needs 20 bars to calculate the first average.

**Solution:** This is expected behavior, not an error. Use `nz()` to replace NaN with 0:
```rhai
nz(sma(close, 20))  // NaN → 0
```

Or start reading after warmup:
```python
# Read only the last 100 bars (skips warmup)
data = read_resource("ct://series/.../tail/100")
```

## "derived series already exists" when materializing indicator

**Cause:** A derived series with the same name already exists.

**Solution:**
1. Use a different name (`name`):
```python
materializar_indicador(fonte="ct://series/...", name="btc_rsi_v2", receita="rsi(close, 14)")
```
2. Or remove the old series first:
```python
remover_serie(uri="ct://derived/btc_rsi")
```

> Back to: [README](./README.en.md)
