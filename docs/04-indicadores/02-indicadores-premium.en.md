# Premium Indicators (17 CT) — License Required

> Proprietary CT-family indicators. Visible and executable **only with a valid premium license**.

Premium indicators live in a separate layer in the `ct-mcp-server`. Without a valid license, they don't appear in `tools/list` and cannot be called. Verification happens in two layers: structural (at server startup) and per-call (on every execution).

---

## CT Family — Balance/Flow/Imbalance

The CT family is based on one principle: **imbalance is the primitive**. Force = `(a − b) / (|a| + |b|)`, signed in `[-1, +1]`. No contest → 0. For each indicator, there's a raw version and a `ct_*` version (VWMA windowed weighted by the indicator's natural size).

| Tool | Input | Window | What it measures | Scale |
|---|---|---|---|---|
| `bop` | OHLC | — | Balance of Power (Livshin) — close vs open force | `[-1, +1]` |
| `ct_bop` | OHLCV | `period` | VWMA of BOP weighted by volume | `[-1, +1]` |
| `tfi` | `trades_1s` | — | Trade Flow Imbalance — `qty_delta / qty` of taker flow | `[-1, +1]` |
| `ct_tfi` | `trades_1s` | `period` | VWMA of TFI weighted by `qty` | `[-1, +1]` |
| `bfi` | `book_1s` | — | Book Flow Imbalance — bid vs ask qty delta | `[-1, +1]` |
| `ct_bfi` | `book_1s` | `period` | VWMA of BFI weighted by `\|bid_delta\| + \|ask_delta\|` | `[-1, +1]` |
| `obi` | `book_1s` | — | Order Book Imbalance top-of-book | `[-1, +1]` |
| `ct_obi` | `book_1s` | `period` | VWMA of OBI weighted by `bid_qty + ask_qty` | `[-1, +1]` |

> **Note:** `tfi`, `bfi`, `obi` and `ct_*` variants require microstructure data (`trades_1s` or `book_1s`) collected via `coletar_trades`/`coletar_book`. See [Microstructure](../07-microestrutura/).

---

## Book microstructure (without trades)

| Tool | Input | What it measures |
|---|---|---|
| `dbi_01` | `book_1s` | Depth Book Imbalance bin ±0.1% (deep version of OBI) |
| `dbi_1` | `book_1s` | Depth Book Imbalance bin ±1% |
| `mpo` | `book_1s` | Microprice Offset normalized by half-spread; bullish skew ↔ `mpo > 0` |

---

## Candle statistical analysis

| Tool | Input | Parameters | What it produces |
|---|---|---|---|
| `ct_candle` | OHLCV | `period`, `short_period` | 28 raw candle-by-candle features (direction/spread/volume + structural) |
| `ct_candle_classificado` | derived from `ct_candle` | 4 parameterizable thresholds | 18 descriptive categorical axes (no school technical naming) |
| `ct_fibo_candle` | OHLCV | `period`, `threshold_volume_z` | Fibonacci projection from anchor candle range (30 columns) |

### `ct_candle` — features produced

- **10 continuous:** direction, body, upper/lower wick, spread, volume, etc.
- **7 comparative (`cmp_*`):** ∈ {-1, 0, +1} — comparison with previous bar
- **4 boolean (`is_*`):** ∈ {0, 1} — new highs/lows in period
- **7 structural:** `close_position_in_range`, `gap_pct_change`, `overlap_ratio`, `is_inside_bar`, `is_outside_bar`, `volume_slope_short`, `spread_slope_short`

### `ct_candle_classificado` — categorical axes

- 8 signed `{-1, 0, +1}`: direction/cmp/gap/trends/acceleration
- 5 ordinal `0..N`: body/wicks/close_position/regimes
- 3 categorical: wick_symmetry, direction_continuity, overlap
- 1 signed 5-state: effort_result_divergence ∈ -2..+2

### `ct_fibo_candle` — what it returns

30 columns: `is_ancora` + 7 Fibonacci level prices (0/50/100/161.8/261.8/-61.8/-161.8) + 7 phases (active/invalid/target) + 7 directions + 7 defenses (wick rejecting level) + `niveis_ativos`.

---

## Proprietary detectors (experimental)

| Tool | Input | Parameters | What it detects |
|---|---|---|---|
| `ct_swing` | OHLCV | `period_ct_bop`, `janela_calibracao`, `quantil_zero`, `quantil_extremo` | Force reversal via `ct_bop` with rolling calibration (empirical quantiles). 7 columns including `sinal_swing` ∈ {-2, -1, 0, +1, +2} |
| `ct_range` | OHLCV | `janela_zscore`, `janela_atr`, `k_mov`, `z_min`, `tau_r` | Range detection from high-volume anchor candle. 4 columns: `range_ativo`, `range_topo`, `range_fundo`, `range_idade` |
| `ct_tendencia` | OHLCV | same params as `ct_range` | Causal trend states after range breakout. 5 columns: `tend_ativa`, `direcao`, `fase` (inativo/impulso/reteste/continuação/armadilha), `progresso`, `nivel_rompido` |
| `ct_momento` | OHLCV | range params + `period` (default 20), `w_tercil` (default 480) | CT momentum with rolling volatility/volume terciles |

### `ct_swing` — sinal_swing

| Value | Meaning |
|---|---|
| -2 | Reversal from high (bearish) |
| -1 | Pullback from high (bearish) |
| 0 | No signal |
| +1 | Pullback from low (bullish) |
| +2 | Reversal from low (bullish) |

### `ct_tendencia` — phases

| Phase | Meaning |
|---|---|
| 0 | Inactive |
| 1 | Impulse |
| 2 | Retest |
| 3 | Continuation |
| 4 | Trap |

> **Important:** "Position is not a verdict" — phase 4 reports regression and may recover. A breakout can turn into a trap → opposite trend, or vice-versa.

---

## Usage example

### `ct_candle` on BTCUSDT 15m

```json
{
  "name": "ct_candle",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "period": 20,
    "short_period": 5
  }
}
```

**Return (summary):**
```json
{
  "uri": "ct://derived/ct_candle_...",
  "value_names": ["direcao", "corpo", "pavio_sup", "pavio_inf", "...28 columns"],
  "valid_from_ts": 1784072400,
  "latest": [...]
}
```

### `ct_tendencia` with custom parameters

```json
{
  "name": "ct_tendencia",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "janela_zscore": 96,
    "janela_atr": 14,
    "k_mov": 3,
    "z_min": 2.0,
    "tau_r": 1.5
  }
}
```

---

## Checking license

```json
// Resource: ct://license/info
// Returns: { licensed: true, plan: "premium", expires_at: "2026-12-31", ... }
```

If the license expires or is invalid, premium tools return an error with renewal instructions.

---

> Next: [Declarative pipeline (DAG)](./03-pipeline-declarativo.en.md) — combine indicators
> Microstructure data: [Trades/book collection](../07-microestrutura/)
