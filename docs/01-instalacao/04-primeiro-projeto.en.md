# 04 — First Project

Congratulations! You've reached the final installation step. Now let's do a
**complete end-to-end walkthrough**: fetch market data, compute an indicator,
run a backtest, and interpret the results.

This document is hands-on — each section shows what you type in the CT Lab chat
and which tools the AI calls behind the scenes.

---

## Table of Contents

- [Project overview](#project-overview)
- [Step 1 — Fetch data (BTCUSDT 15m)](#step-1--fetch-data-btcusdt-15m)
- [Step 2 — Compute RSI](#step-2--compute-rsi)
- [Step 3 — Simple backtest](#step-3--simple-backtest)
- [Step 4 — Interpret results](#step-4--interpret-results)
- [Bonus: using Python](#bonus-using-python)

---

## Project overview

Our example strategy will be simple:

> **"If today's closing price is higher than yesterday's, buy.
> Otherwise, go flat."**

For this, we'll need:

1. **Market data** — BTCUSDT candles at 15m from Binance.
2. **Indicator** — RSI with period 14 to track momentum.
3. **Backtest** — simulate the strategy with $1,000 capital.

---

## Step 1 — Fetch data (BTCUSDT 15m)

### In the CT Lab chat, type:

> **"Fetch the BTCUSDT series at 15m interval from Binance."**

### What the AI does behind the scenes:

The AI calls the `buscar_serie` (JSON-RPC) / `buscarSerie` (TypeScript SDK) tool,
which internally uses `buscar_binance` / `buscarBinance`:

**JSON-RPC (MCP):**
```json
{
  "method": "tools/call",
  "params": {
    "name": "buscar_serie",
    "arguments": {
      "provider": "binance",
      "symbol": "BTCUSDT",
      "interval": "15m"
    }
  }
}
```

**TypeScript SDK:**
```typescript
const result = await Ct.buscarSerie({
  provider: "binance",
  symbol: "BTCUSDT",
  interval: "15m"
});
```

### Expected response:

```text
🤖 Series loaded successfully!

   • URI: ct://series/binance/BTCUSDT/15m
   • Candles loaded: 500
   • First candle: 2026-08-06 00:00 UTC
   • Last candle: 2026-08-11 15:45 UTC
```

The **URI** (`ct://series/binance/BTCUSDT/15m`) is the unique identifier for
the series. You'll use it in the next steps to reference the data.

> 💡 The URI is permanent: even if you close CT Lab, the series stays in cache
> and can be reused.

---

## Step 2 — Compute RSI

### In the CT Lab chat, type:

> **"Calculate the 14-period RSI for this series."**

### What the AI does behind the scenes:

The AI uses the URI from the previous series and calls the `rsi` tool:

**JSON-RPC (MCP):**
```json
{
  "method": "tools/call",
  "params": {
    "name": "rsi",
    "arguments": {
      "uri": "ct://series/binance/BTCUSDT/15m",
      "period": 14
    }
  }
}
```

**TypeScript SDK:**
```typescript
const result = await Ct.rsi({
  uri: "ct://series/binance/BTCUSDT/15m",
  period: 14
});
```

### Expected response:

```text
🤖 RSI(14) computed for BTCUSDT 15m.

   • Indicator URI: ct://derived/binance/BTCUSDT/15m/rsi_14
   • Value names: ["rsi"]
   • Latest value: 58.32
   • Valid from: 2026-08-06 03:30 UTC
```

RSI ranges from 0 to 100:

- **> 70**: overbought market (possible downward reversal)
- **< 30**: oversold market (possible upward reversal)
- **30–70**: neutral zone

> 📊 The indicator also stays in cache. If you request it again later, the
> response is instant.

---

## Step 3 — Simple backtest

### In the CT Lab chat, type:

> **"Run a backtest with the following strategy: if the current close is
> higher than the previous close, go long; otherwise, go flat. Use $1,000
> capital and 0.1% fee per trade."**

### What the AI does behind the scenes:

The AI assembles the Rhai script and calls the `ct_backtest` (JSON-RPC) /
`ctBacktest` (TypeScript SDK) tool:

**JSON-RPC (MCP):**
```json
{
  "method": "tools/call",
  "params": {
    "name": "ct_backtest",
    "arguments": {
      "serie": "ct://series/binance/BTCUSDT/15m",
      "estrategia_script": "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
      "capital_inicial": 1000,
      "fee_pct": 0.001,
      "nome": "exemplo_momentum"
    }
  }
}
```

**TypeScript SDK:**
```typescript
const result = await Ct.ctBacktest({
  serie: "ct://series/binance/BTCUSDT/15m",
  estrategiaScript: "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
  capitalInicial: 1000,
  feePct: 0.001,
  nome: "exemplo_momentum"
});
```

### Rhai syntax for backtests

The strategy is written in **Rhai**, a safe embedded scripting language in
ct-mcp-server:

| Function | Meaning |
|----------|---------|
| `comprado(1.0)` | Go **100% long**. The value must be a float. |
| `vendido(1.0)` | Go **100% short**. The value must be a float. |
| `zerado()` | Go **100% flat** (no position). |
| `close[0]` | Closing price of the **current** candle. |
| `close[1]` | Closing price of the **previous** candle. |
| `ind["alias"][0]` | Current value of an indicator with alias. |

> ⚠️ **Important**: always use `float` in arguments: `comprado(1.0)`, not
> `comprado(1)`. Rhai is typed and requires f64.

---

## Step 4 — Interpret results

### Expected response:

```text
🤖 Backtest complete — "exemplo_momentum"

   📈 Performance metrics:
   ┌────────────────────────────────────────────────┐
   │  Total return:           +12.34%                │
   │  Total PnL:              +$123.40               │
   │  Gross PnL:              +$145.20               │
   │  Total fees:             -$21.80                │
   │  Number of trades:       87                     │
   │  Win rate:               54.0%                  │
   │  Profit factor:          1.32                   │
   │  Max drawdown:           -8.45%                 │
   │  Sharpe ratio:           1.21                   │
   │  Sortino ratio:          1.68                   │
   │  Calmar ratio:           1.46                   │
   └────────────────────────────────────────────────┘

   • Equity URI: ct://backtest/exemplo_momentum/equity
   • Trades URI: ct://backtest/exemplo_momentum/trades
```

### Interpretation guide

| Metric | What it means | Reference value |
|--------|---------------|-----------------|
| **Total return** | Percentage gain/loss over the period | > 0 is good |
| **Win rate** | % of profitable trades | > 50% is reasonable |
| **Profit factor** | Gains ÷ losses | > 1.0 is profitable |
| **Sharpe** | Risk-adjusted return | > 1.0 is good, > 2.0 is great |
| **Sortino** | Return adjusted for downside volatility | Higher than Sharpe = good |
| **Calmar** | Return ÷ max drawdown | > 1.0 is desirable |
| **Max drawdown** | Largest equity drop | Lower (absolute) is better |
| **Total fees** | Transaction costs | Depends on number of trades |

> 💡 This strategy is **educational** and not optimized. In a real project,
> you'd combine multiple indicators (e.g., RSI + SMA) to filter false signals.

---

## Bonus: using Python

If you prefer scripting directly, you can use CT Lab via Python with `uv`:

```bash
# Install dependencies
uv pip install ct-mcp-client

# Create the first project script
cat > first_project.py << 'EOF'
import asyncio
from ct_mcp_client import Client

async def main():
    client = Client()

    # Step 1: fetch series
    serie = await client.call("buscar_serie", {
        "provider": "binance",
        "symbol": "BTCUSDT",
        "interval": "15m"
    })
    print(f"Series: {serie['uri']} ({serie['row_count']} candles)")

    # Step 2: compute RSI
    rsi = await client.call("rsi", {
        "uri": serie["uri"],
        "period": 14
    })
    print(f"Current RSI: {rsi['latest'][0]:.2f}")

    # Step 3: backtest
    bt = await client.call("ct_backtest", {
        "serie": serie["uri"],
        "estrategia_script": "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
        "capital_inicial": 1000,
        "fee_pct": 0.001,
        "nome": "exemplo_momentum"
    })
    print(f"Return: {bt['retorno_total']:.2%}")
    print(f"Sharpe: {bt['sharpe']:.2f}")

asyncio.run(main())
EOF

# Run
uv run first_project.py
```

> The Python API uses `snake_case` for tool names, matching JSON-RPC.

---

## Next Steps

You now have everything set up and working! Explore more:

| Topic | How to explore |
|-------|----------------|
| **More indicators** | Ask: "Compute SMA, MACD, and Bollinger for BTCUSDT." |
| **Complex strategies** | Combine indicators in the Rhai script using `ind["alias"][0]`. |
| **Multiple series** | Fetch ETHUSDT, SOLUSDT and compare. |
| **Optimization** | Use `otimizar_hiperparametros` to find the best parameters. |
| **ML** | If you have Premium, explore `montar_esteira_ml`. |

- ⬅️ **[03 — MCP Connection](./03-conexao-mcp)** — Back
- ⬅️ **[Installation Index](./README)**

---

> 🎉 **Done!** You've completed the CT Lab installation and run your first
> end-to-end project. Happy trading!

_Last updated: 2026-08-11_
