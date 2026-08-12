# 07 · Live Catalogs

CT Lab maintains **live catalogs** — special resources that return the complete list of available components in real time. Unlike static documentation, which ages, catalogs **always reflect the current state** of the system, including new, licensed, and private components.

---

## Why Catalogs Matter

Traditional documentation has a problem: it **ages**. A README listing "5 available indicators" becomes stale when a 6th is added. The user reads outdated docs and misses features.

CT Lab's catalogs solve this:

| Characteristic | Static Docs | Live Catalogs |
|---------------|:-----------:|:-------------:|
| Always current | ❌ | ✅ |
| Includes licensed | ❌ | ✅ (if licensed) |
| Includes private | ❌ | ✅ (if licensed) |
| Versioned with product | ❌ | ✅ |
| AI model reads directly | ❌ | ✅ |

> The AI model can query `ct://indicators/catalog` at any time to know which indicators are available **now**, without relying on potentially outdated documentation.

---

## The Four Catalogs

### 1. `ct://sources/catalog` — Data Sources

Lists all available data providers.

#### AI Chat Prompt

> "What data sources are available in CT Lab?"

#### Resource URI

```
ct://sources/catalog
```

#### Expected Response

```json
{
  "uri": "ct://sources/catalog",
  "sources": [
    {
      "provider": "binance",
      "name": "Binance",
      "description": "Cryptocurrency exchange",
      "supported_intervals": ["1m", "5m", "15m", "1h", "4h", "1d"],
      "features": ["spot", "futures"],
      "requires_api_key": false
    },
    {
      "provider": "yahoo",
      "name": "Yahoo Finance",
      "description": "Stocks, ETFs, indices",
      "supported_intervals": ["1d", "1wk", "1mo"],
      "features": ["stocks", "etf", "index"],
      "requires_api_key": false
    }
  ]
}
```

### 2. `ct://indicators/catalog` — Technical Indicators

Lists all available technical indicators (public + private if licensed).

#### AI Chat Prompt

> "List all available technical indicators."

#### Resource URI

```
ct://indicators/catalog
```

#### Expected Response

```json
{
  "uri": "ct://indicators/catalog",
  "indicators": [
    {
      "name": "rsi",
      "display_name": "Relative Strength Index",
      "category": "momentum",
      "params": { "period": { "type": "number", "default": 14, "min": 1 } },
      "private": false
    },
    {
      "name": "macd",
      "display_name": "MACD",
      "category": "momentum",
      "params": {
        "fast": { "type": "number", "default": 12 },
        "slow": { "type": "number", "default": 26 },
        "signal": { "type": "number", "default": 9 }
      },
      "private": false
    },
    {
      "name": "bollinger",
      "display_name": "Bollinger Bands",
      "category": "volatility",
      "params": {
        "period": { "type": "number", "default": 20 },
        "std_dev": { "type": "number", "default": 2.0 }
      },
      "private": false
    }
  ]
}
```

> Indicators with `private: true` only appear if you have the corresponding license. This prevents the AI model from trying to use an unlicensed indicator.

### 3. `ct://pipeline/catalog` — Pipeline Operations

Lists operations available for building processing pipelines.

#### AI Chat Prompt

> "What pipeline operations can I use?"

#### Resource URI

```
ct://pipeline/catalog
```

#### Expected Response

```json
{
  "uri": "ct://pipeline/catalog",
  "operations": [
    {
      "name": "window",
      "description": "Sliding window for incremental computation",
      "params": { "size": { "type": "number", "required": true } }
    },
    {
      "name": "normalize",
      "description": "Min-max or z-score normalization",
      "params": {
        "method": { "type": "string", "enum": ["minmax", "zscore"], "default": "minmax" }
      }
    },
    {
      "name": "lag",
      "description": "Shifts the series by N periods",
      "params": { "periods": { "type": "number", "default": 1 } }
    }
  ]
}
```

### 4. `ct://ml/catalog` — Machine Learning Components

Lists ML components available for building pipelines and learning workflows.

#### AI Chat Prompt

> "What machine learning components are available?"

#### Resource URI

```
ct://ml/catalog
```

#### Expected Response

```json
{
  "uri": "ct://ml/catalog",
  "components": [
    {
      "name": "linear_regression",
      "display_name": "Linear Regression",
      "category": "regression",
      "params": {
        "fit_intercept": { "type": "boolean", "default": true }
      },
      "private": false
    },
    {
      "name": "random_forest",
      "display_name": "Random Forest Regressor",
      "category": "regression",
      "params": {
        "n_estimators": { "type": "number", "default": 100 },
        "max_depth": { "type": "number", "default": null }
      },
      "private": false
    }
  ]
}
```

---

## How the AI Model Uses Catalogs

The typical AI model flow:

```
1. Reads ct://indicators/catalog     → discovers "rsi" exists
2. Confirms parameters               → period=14, etc.
3. Calls Ct.rsi({ uri, period: 14 })  → computes the indicator
4. Reads result via resource          → analyzes and interprets
```

Without the catalog, the model would have to guess names and parameters — leading to errors. With the catalog, the model discovers **programmatically** what's available.

---

## Benefits Summary

| Benefit | Explanation |
|---------|-------------|
| **Always current** | Catalogs are generated dynamically, never hardcoded |
| **No stale docs** | Doesn't depend on manually updated markdown |
| **Automatic discovery** | AI models discover new components without human prompting |
| **License filtering** | Private components only appear for licensed users |
| **Parameter schemas** | Each component lists its parameters and types |

---

## TypeScript Example

```typescript
// Discover all available indicators
const catalogResource = await Ct.readResource("ct://indicators/catalog");
const indicators = catalogResource.indicators;
console.log(`${indicators.length} indicators available:`);
indicators.forEach(ind => {
  const visibility = ind.private ? "🔒 private" : "🌐 public";
  console.log(`  ${ind.display_name} (${ind.name}) — ${ind.category} ${visibility}`);
});

// Check data sources
const sourcesCatalog = await Ct.readResource("ct://sources/catalog");
sourcesCatalog.sources.forEach(src => {
  console.log(`  ${src.name}: ${src.supported_intervals.join(", ")}`);
});
```

---

## Python Example with `uv`

```python
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # List available indicators
    catalog = await client.read_resource("ct://indicators/catalog")
    print("Available indicators:")
    for ind in catalog["indicators"]:
        LOCK = "🔒" if ind.get("private") else "🌐"
        print(f"  {LOCK} {ind['display_name']} ({ind['name']})")

    # Check pipeline ops
    ops = await client.read_resource("ct://pipeline/catalog")
    print(f"\nPipeline operations: {len(ops['operations'])}")
    for op in ops["operations"]:
        print(f"  - {op['name']}: {op['description']}")

asyncio.run(main())
```

```bash
uv run main.py
```

---

[← Back to category 03](README.en.md)
