# Declarative Pipeline (DAG)

> `montar_pipeline_indicadores` — chain indicators and ops in a declarative acyclic graph (DAG). The most powerful way to compose signals.

The pipeline lets you execute a sequence of steps where each step references prior steps via `$id`. Only the final step (`output`) is persisted as a derived series. Intermediaries stay in memory — they don't pollute the cache.

---

## Pipeline anatomy

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "my_signal",
    "output": "$signal",
    "steps": [ ... ]
  }
}
```

| Field | What it is |
|---|---|
| `anchor` | Raw URI that anchors the entire pipeline. Defines the timeline. |
| `name` | Final series name: `ct://derived/<name>` |
| `output` | `$<id>` reference to the step whose output becomes the persisted series |
| `steps` | List of steps in topological order |

Each step has:
- `id` — unique identifier (referenced by other steps via `$id`)
- `op` — what to do (indicator, combine, transform, etc.)
- `source` — input: `$anchor`, `$<id>` (prior step), or a URI

---

## Available ops

Always consult the live catalog:

**Resource:** `ct://pipeline/catalog`

### Indicators (same 53 tools)
Each indicator is a pipeline op. Example: `{ "id": "my_rsi", "op": "rsi", "source": "$anchor", "period": 14 }`

### Declarative ops

| Op | What it does | Example |
|---|---|---|
| `combinar_aritmetica` | `+ − × ÷` between N operands (series or scalars) | Add RSI + MFI |
| `comparar` | Relation/crossover series×series → 0/1 | `cruza_acima`, `cruza_abaixo`, `maior`, `menor` |
| `condicional` | Ternary: if condition → then X else Y | `if rsi > 30 then 1 else 0` |
| `transformar` | `abs`, `log`, `sqrt`, `clamp`, `sinal`, `neg` | Log of volume |
| `estatistica_rolling` | `rma`, `smm`, `desvio_padrao`, `regressao_linear` rolling window | 20-bar standard deviation |
| `compose` | Inner-join of steps by timestamp (cross-series) | BTC + ETH aligned |
| `custom` | Rhai or Python script inline/uri | Custom logic |

> **Caveat:** `comparar` is **series×series**. Threshold against a scalar (`rsi < 30`) is `condicional` or `custom`, not `comparar`.

---

## Full example: RSI×SMA crossover signal

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "rsi_sma_signal",
    "output": "$signal",
    "steps": [
      { "id": "rsi", "op": "rsi", "source": "$anchor", "period": 14 },
      { "id": "sma_rsi", "op": "sma", "source": "$rsi", "period": 9 },
      {
        "id": "cross_above",
        "op": "comparar",
        "esquerda": "$rsi",
        "direita": "$sma_rsi",
        "operador": "cruza_acima"
      },
      {
        "id": "cross_below",
        "op": "comparar",
        "esquerda": "$rsi",
        "direita": "$sma_rsi",
        "operador": "cruza_abaixo"
      },
      {
        "id": "signal",
        "op": "condicional",
        "condicao": "$cross_above",
        "entao": { "escalar": 1.0 },
        "senao": { "escalar": -1.0 },
        "coluna_saida": "signal"
      }
    ]
  }
}
```

**Return:**
```json
{
  "uri": "ct://derived/rsi_sma_signal",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "steps_executed": 5
}
```

---

## When to use pipeline vs Rhai vs direct tool

| Via | When to use |
|---|---|
| **Direct tool** (`sma`, `rsi`, etc.) | 1 simple indicator, no combination |
| **Vectorized Rhai** (`materializar_indicador`) | Single expression chaining indicators: `sma(rsi(close, 14), 5)` |
| **Pipeline (DAG)** | 4+ steps, reusable declarative logic, multiple branches |
| **Compose** (`compor_serie`) | Join N series from **different assets** by timestamp |

> **Rule:** pipeline is the only via that supports trees (multiple branches converging). Rhai is linear (one expression). Direct tool is a single computation.

---

> Next: [Declarative ops — details](./04-ops-declarativas.en.md) · [Vectorized Rhai](./05-rhai-vetorizado.en.md) · [Cookbook](./07-cookbook.en.md)
