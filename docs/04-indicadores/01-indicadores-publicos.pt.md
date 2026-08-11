# Indicadores Built-in (36 Públicos)

> Catálogo completo de indicadores técnicos disponíveis sem licença premium.

O `ct-mcp-server` expõe **36 indicadores públicos** baseados na biblioteca [yata](https://crates.io/crates/yata) `0.7.0`, mais 2 customs (`atr`, `obv`). Todos seguem o mesmo padrão de chamada e retorno.

---

## Padrão de chamada

Toda tool de indicador recebe uma URI de série e parâmetros próprios, e retorna **URI + meta** (nunca rows):

```json
{
  "name": "rsi",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "period": 14
  }
}
```

**Retorno:**
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

| Campo | Significado |
|---|---|
| `uri` | URI persistida da série derivada (`ct://derived/<name>`) |
| `valid_from_ts` | Onde o indicador deixa de ser NaN (warmup awareness) |
| `value_names` | Nomes das colunas produzidas |
| `latest` | Últimos valores calculados (alinhados com `value_names`) |
| `evicted_series` | Séries removidas do cache (LRU) para abrir espaço |

> **Dica:** o nome da derived é auto-gerado (`<ind>_<col>_<params>_<hash6>`) se você não passar `name`. Passe `name` para um nome semântico (`"btc_rsi14"`).

---

## Catálogo por categoria

### Tendência (médias móveis)

| Tool | Indicador | Parâmetros | Colunas de saída |
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

| Tool | Indicador | Parâmetros | Colunas de saída |
|---|---|---|---|
| `rsi` | Relative Strength Index | `period` | `rsi` |
| `macd` | MACD | (fixo 12,26,9) | `macd`, `signal`, `hist` |
| `stochastic` | Stochastic Oscillator | (fixo 14,3) | `k`, `d` |
| `momentum_idx` | Momentum Index | (default 10,1) | `momentum` |
| `cci` | Commodity Channel Index | `period` | `cci` |
| `cmo` | Chande Momentum Oscillator | `period` | `cmo` |
| `williams_r` | Williams %R | `period` | `williams_r` |
| `trix` | TRIX | `period` | `trix` |
| `tsi` | True Strength Index | (default 25,13,13) | `tsi` |
| `roc` | Rate of Change | `period` | `roc` |
| `coppock` | Coppock Curve | (default 14,11,10) | `coppock` |
| `dpo` | Detrended Price Oscillator | (default 21) | `dpo` |
| `fisher` | Fisher Transform | `period` | `fisher` |
| `awesome` | Awesome Oscillator | (default 5,34) | `awesome` |
| `smi` | SMI Ergodic Indicator | (default 20,5,5) | `smi` |
| `rvi` | Relative Vigor Index | `period` | `rvi` |
| `kst` | Know Sure Thing | (default) | `kst`, `signal` |
| `woodies` | Woodies CCI | (default 6,14) | `cci`, `signal` |

### Volatilidade

| Tool | Indicador | Parâmetros | Colunas de saída |
|---|---|---|---|
| `atr` | Average True Range | `period` | `atr` |
| `bollinger` | Bollinger Bands | (fixo 20, 2σ) | `upper`, `middle`, `lower` |
| `donchian` | Donchian Channel | `period` | `upper`, `lower` |
| `keltner` | Keltner Channel | (default) | `upper`, `middle`, `lower` |
| `envelopes` | Moving Envelopes | (default SMA20, k=0.1) | `upper`, `lower` |
| `chande_kroll` | Chande Kroll Stop | (default) | `stop_long`, `stop_short` |
| `price_channel` | Price Channel | `period` | `upper`, `lower` |

### Volume

| Tool | Indicador | Parâmetros | Colunas de saída |
|---|---|---|---|
| `obv` | On-Balance Volume | — | `obv` |
| `cmf` | Chaikin Money Flow | `period` | `cmf` |
| `mfi` | Money Flow Index | `period` | `mfi` |
| `efi` | Elder's Force Index | (default) | `efi` |
| `eom` | Ease of Movement | (default) | `eom` |
| `klinger` | Klinger Volume Oscillator | (default 34,55) | `kvo`, `signal` |
| `chaikin_osc` | Chaikin Oscillator | (default 3,10) | `chaikin_osc` |

### Trend Strength

| Tool | Indicador | Parâmetros | Colunas de saída |
|---|---|---|---|
| `adx` | Average Directional Index | (default) | `adx`, `di_plus`, `di_minus` |
| `trend_strength` | Trend Strength Index | `period` | `tsi` |

---

## Smart defaults de coluna

Para indicadores de coluna única (`sma`, `ema`, `rsi`, etc.), se você omitir `column`, o sistema usa:
- `"close"` se existir na série
- A única coluna disponível, se houver apenas uma
- Erro se há múltiplas colunas e nenhuma é `"close"`

Para indicadores multi-coluna (`atr`, `bollinger`, `ichimoku`, etc.), os defaults são `"high"`, `"low"`, `"close"`, `"volume"` conforme apropriado.

---

## Exemplos práticos

### RSI(14) no BTCUSDT 15m

**Chat:**
> Calcule RSI de 14 períodos no BTCUSDT 15m

```json
{ "name": "rsi", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 14 } }
```

### MACD (12,26,9) com nome custom

**Chat:**
> Calcule MACD no BTCUSDT 15m e chame de "btc_macd"

```json
{ "name": "macd", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "name": "btc_macd" } }
```

**Retorno:**
```json
{
  "uri": "ct://derived/btc_macd",
  "value_names": ["macd", "signal", "hist"],
  "latest": [12.34, 10.5, 1.84]
}
```

### Bollinger Bands (3 colunas)

```json
{ "name": "bollinger", "arguments": { "uri": "ct://series/binance/BTCUSDT/15m" } }
```

**Retorno:**
```json
{
  "uri": "ct://derived/bollinger_close_20_2sigma_abc123",
  "value_names": ["upper", "middle", "lower"],
  "latest": [64200.0, 63700.0, 63200.0]
}
```

---

## Chain walking (indicador sobre indicador)

Todas as tools de indicador aceitam uma URI **derived** como entrada (não apenas raw). O sistema segue a cadeia `source_uris` recursivamente até achar a raw original:

```json
{ "name": "ema", "arguments": { "uri": "ct://derived/btc_macd", "period": 9 } }
```

Isto calcula EMA(9) sobre a coluna `"close"` da derived do MACD. Para escolher outra coluna, passe `column`:

```json
{ "name": "ema", "arguments": { "uri": "ct://derived/btc_macd", "period": 9, "column": "hist" } }
```

---

## Catálogo vivo

A lista acima pode desatualizar. Para ver sempre a versão atual:

**Chat:**
> Liste todos os indicadores disponíveis

O modelo lê o resource `ct://indicators/catalog` e retorna a lista completa com parâmetros e colunas de saída.

---

> **Indicadores premium (CT)?** Veja [02-indicadores-premium](./02-indicadores-premium.pt.md).
> **Combinar indicadores?** Veja [Pipeline declarativo](./03-pipeline-declarativo.pt.md) e [Rhai vetorizado](./05-rhai-vetorizado.pt.md).
