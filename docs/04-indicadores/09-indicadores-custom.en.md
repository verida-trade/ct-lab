# Custom Indicators (Rhai and Python)

> Create fully custom indicators using Rhai or Python scripts inside the pipeline.

The `custom` step in the pipeline allows running arbitrary logic — either in Rhai (embedded language, no dependencies) or Python (via `uv`, with libraries).

---

## Rhai custom (inline)

Faster — no external environment, runs directly in the server process.

```json
{
  "id": "my_signal",
  "operacao": "custom",
  "script": "let r = rsi(close, par[\"p\"]); let m = sma(close, par[\"m\"]); if r > 70.0 && close[0] > m[0] { -1.0 } else if r < 30.0 && close[0] < m[0] { 1.0 } else { 0.0 }",
  "entradas": [
    { "alias": "close", "fonte": "$anchor", "coluna": "close" },
    { "alias": "high", "fonte": "$anchor", "coluna": "high" },
    { "alias": "low", "fonte": "$anchor", "coluna": "low" }
  ],
  "parametros": { "p": 14, "m": 20 },
  "coluna_saida": "signal"
}
```

### How it works

- `entradas` maps alias → series/column. In the script, access via `ent["alias"]` or `close`, `high`, etc. (if alias matches native column name).
- `parametros` exposes values as `par["name"]` in the script.
- `script` is a Rhai expression. To return multiple columns, use a map: `#{ "col1": ..., "col2": ... }`.

### Rhai via URI

Instead of inline, reference a `.rhai` file:

```json
{
  "id": "custom",
  "operacao": "custom",
  "uri": "file:///path/to/my_indicator.rhai",
  "entradas": [...],
  "parametros": {...}
}
```

---

## Python custom (inline)

For logic that needs Python libraries (numpy, pandas, scipy, etc.):

```json
{
  "id": "zscore_py",
  "operacao": "custom",
  "script": "import numpy as np\n\ndef calcular(close, high, low, volume, par):\n    r = np.array(close)\n    m = np.mean(r)\n    s = np.std(r)\n    z = (r - m) / s if s > 0 else np.zeros_like(r)\n    return {\"zscore\": z.tolist()}",
  "entradas": [
    { "alias": "close", "fonte": "$anchor", "coluna": "close" }
  ],
  "parametros": {}
}
```

### Python environment

The `ct-mcp-server` runs Python via `uv` in a controlled ephemeral environment:

- **Fixed version:** `CT_MCP_ML_PYTHON` (default `3.14.5`)
- **Deps:** base libraries per family (`gbdt` → scikit-learn, `mlp` → torch). For simple custom scripts, `numpy` comes in the base environment.
- **Extra deps:** pass `deps: ["scipy==1.14.0"]` in the step to add libraries.
- **No fallback:** if `uv` is not installed, the server returns an error with installation instructions — it does not degrade to host Python.

```json
{
  "id": "custom_scipy",
  "operacao": "custom",
  "script": "from scipy.signal import find_peaks\n\ndef calcular(close, par):\n    peaks, _ = find_peaks(close, distance=par[\"dist\"])\n    result = [0.0] * len(close)\n    for p in peaks:\n        result[p] = 1.0\n    return {\"peaks\": result}",
  "deps": ["scipy==1.14.0"],
  "entradas": [{ "alias": "close", "fonte": "$anchor", "coluna": "close" }],
  "parametros": { "dist": 5 }
}
```

### Python via URI

```json
{
  "id": "custom",
  "operacao": "custom",
  "uri": "file:///path/to/my_indicator.py",
  "deps": ["pandas==2.2"],
  "entradas": [...]
}
```

---

## Python script contract

The Python script must define a `calcular` function that takes inputs as Python arrays (lists) and returns a dict `{ "column": [values] }`:

```python
def calcular(close, high, low, volume, par):
    # par is a dict with parameters
    # close, high, low, volume are lists of floats
    result = [c * par.get("factor", 1.0) for c in close]
    return {"custom_col": result}
```

---

## When to use Rhai vs Python

| Criterion | Rhai | Python |
|---|---|---|
| Speed | ⚡ Faster (in-process) | ⏳ `uv` startup overhead |
| Dependencies | None (embedded) | Any Python lib via `uv` |
| Complexity | Simple/medium expressions | Complex math, matrices, ML |
| State | Stateless (per expression) | Stateless (per call) |
| Best for | Signals, filters, algebra | Statistics, ML, complex transforms |

---

> Next: [Declarative pipeline](./03-pipeline-declarativo.en.md) · [Vectorized Rhai](./05-rhai-vetorizado.en.md)
