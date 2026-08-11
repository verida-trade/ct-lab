# Free vs Premium Plans

> **Folder:** `docs/02-conceitos/06-free-vs-premium.en.md`  
> **Related reading:** [`01-visao-geral`](./01-visao-geral.en.md) ·
> [`05-mcp-protocolo`](./05-mcp-protocolo.en.md)  
> **Target audience:** all levels

---

## Overview

CT Lab operates with two plans: **Free** and **Premium**. The difference
between them is not just resource quantity — it's also the type of analytical
capability available. The Free plan covers basic data analysis and backtesting;
Premium unlocks the full arsenal of proprietary CT indicators, ML,
microstructure, and robustness testing.

Licensing is **local** — validated via the `ct://license/info` URI on
`ct-labd`.

---

## Comparison Table

| Resource | Free | Premium |
|----------|------|---------|
| **Series cache** | 1 series | 100 series |
| **Public indicators** (36) | ✅ SMA, EMA, RSI, MACD, Bollinger, ATR, … | ✅ All 36 |
| **CT indicators** (17) | ❌ | ✅ ct_bop, ct_tfi, ct_bfi, ct_obi, ct_candle, ct_swing, ct_range, ct_tendencia, ct_fibo_candle, ct_candle_classificado, ct_momento, bop, tfi, bfi, obi, dbi_01, dbi_1, mpo |
| **Basic backtest** | ✅ `ct_backtest` | ✅ |
| **Search backtests** | ✅ `ct_buscar_backtests` | ✅ |
| **ML pipeline** | ❌ | ✅ `montar_esteira_ml`, `aplicar_modelo`, `otimizar_hiperparametros` |
| **Microstructure** | ❌ | ✅ `coletar_book`, `coletar_trades`, `consultar_book`, `consultar_trades` |
| **Kline collection** | ✅ `coletar_klines`, `parar_klines` | ✅ |
| **Survival test** | ❌ | ✅ `ct_testar_sobrevivencia` |
| **Libraries (libs)** | ❌ | ✅ `salvar_lib`, `ler_lib`, `listar_libs`, `excluir_lib` |
| **Filters** | ❌ | ✅ `salvar_filtro`, `listar_filtros`, `excluir_filtro` |
| **ML models** | ❌ | ✅ `listar_modelos`, `excluir_modelo` |
| **Public prompts** | ✅ `saudacao`, `comecar`, `backtest` | ✅ |
| **Premium prompts** | ❌ | ✅ `coleta`, `esteira`, `fundacao`, `regime` |
| **Discovery** | ✅ `buscar_serie`, `listar_series`, `top_ativos` | ✅ |
| **CSV import** | ✅ `importar_csv` | ✅ |
| **Billing** | ✅ `comprar_premium`, `cancelar_assinatura` | ✅ |

---

## The 17 CT Indicators (Premium)

These are the proprietary indicators exclusive to the Premium plan:

| # | Tool (MCP) | SDK (TS) | Category |
|---|-----------|----------|----------|
| 1 | `ct_bop` | `ctBop` | Balance of Power |
| 2 | `ct_tfi` | `ctTfi` | Trend Flow Index |
| 3 | `ct_bfi` | `ctBfi` | Buying Force Index |
| 4 | `ct_obi` | `ctObi` | Order Book Imbalance |
| 5 | `ct_candle` | `ctCandle` | Candle Analysis |
| 6 | `ct_swing` | `ctSwing` | Swing Analysis |
| 7 | `ct_range` | `ctRange` | Range Analysis |
| 8 | `ct_tendencia` | `ctTendencia` | Trend Analysis |
| 9 | `ct_fibo_candle` | `ctFiboCandle` | Fibonacci Candle |
| 10 | `ct_candle_classificado` | `ctCandleClassificado` | Classified Candle |
| 11 | `ct_momento` | `ctMomento` | Momentum Analysis |
| 12 | `bop` | `bop` | Balance of Power (legacy) |
| 13 | `tfi` | `tfi` | Trend Flow Index (legacy) |
| 14 | `bfi` | `bfi` | Buying Force Index (legacy) |
| 15 | `obi` | `obi` | Order Book Imbalance (legacy) |
| 16 | `dbi_01` | `dbi01` | Delta Balance Index 01 |
| 17 | `dbi_1` | `dbi1` | Delta Balance Index 1 |

> The 36 public indicators are documented in `docs/03-indicadores/`.

---

## How License Gating Works

The license gate is enforced in `ct-labd`. When the AI tries to invoke a
Premium tool with a Free license, the call returns a structured error:

```json
{
  "error": "premium_required",
  "tool": "ct_bop",
  "message": "This tool requires the Premium plan.",
  "upgrade_uri": "ct://license/upgrade"
}
```

The LLM, upon receiving this error, can inform the user and suggest upgrading.

### Diagram

```
  AI invokes Premium tool (e.g., ct_bop)
       │
       ▼
  ct-mcp-server ──► ct-labd
       │                 │
       │          ┌──────┴──────┐
       │          ▼             ▼
       │    License OK     License Free/Unregistered
       │    (Premium)       │
       │          │             │
       │          ▼             ▼
       │    Executes tool    Returns error
       │    normally         "premium_required"
       │          │             │
       └──────────┴─────────────┘
                       │
                       ▼
              LLM informs the user
```

---

## How to Check License Status

The `ct://license/info` URI returns the current license information:

```python
# Check license status
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "info = read_resource('ct://license/info'); "
    "print(info)"
], capture_output=True, text=True)
print(result.stdout)
```

### Example Responses

**Free plan:**
```json
{
  "plan": "free",
  "premium": false,
  "max_cache": 1,
  "indicators_public": 36,
  "indicators_ct": 0,
  "ml_enabled": false
}
```

**Premium plan:**
```json
{
  "plan": "premium",
  "premium": true,
  "max_cache": 100,
  "indicators_public": 36,
  "indicators_ct": 17,
  "ml_enabled": true,
  "valid_until": "2026-12-31"
}
```

---

## How to Upgrade

### Using the MCP tool `comprar_premium`

```typescript
// TypeScript SDK: initiate the upgrade process
const result = await Ct.comprarPremium({
    // billing parameters as needed
});
// → Returns payment/subscription information
```

### Using Python

```python
# Initiate the upgrade to Premium
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import comprar_premium; "
    "result = comprar_premium(); "
    "print(result)"
], capture_output=True, text=True)
print(result.stdout)
# → Returns checkout/subscription data
```

### Cancel subscription

```python
# Cancel the Premium subscription
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import cancelar_assinatura; "
    "result = cancelar_assinatura(); "
    "print(result)"
], capture_output=True, text=True)
print(result.stdout)
```

---

## Tool Visibility Tiers

CT Lab classifies tools into 3 visibility levels:

| Level | Environment variable | Description |
|-------|---------------------|-------------|
| **public** | (always visible) | 36 indicators, data, backtest, billing, basic prompts |
| **private** | Premium license | 17 CT indicators, ML, microstructure, libs, premium prompts |
| **diag** | `CT_MCP_DIAG` | Developer-only tools — not for end users |

> `diag` tools only appear when the `CT_MCP_DIAG` environment variable is set.
> They should not be used by end users.

---

## FAQ

| Question | Answer |
|----------|--------|
| Do I need internet for the Free plan? | Yes — to fetch data from providers. The cache is local. |
| Does Premium expire? | Yes, per `valid_until` in `ct://license/info`. |
| Can I use ML on the Free plan? | No. The entire ML pipeline is Premium. |
| Is backtest Free? | Yes, basic backtest (`ct_backtest`) is Free. |
| How many series can I cache? | Free: 1. Premium: 100. |

---

## Next Steps

- [`01-visao-geral`](./01-visao-geral.en.md) — ecosystem overview
- [`05-mcp-protocolo`](./05-mcp-protocolo.en.md) — tools, resources, prompts
- `docs/03-indicadores/` — documentation of the 36 public indicators
