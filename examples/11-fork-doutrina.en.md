# 11 — Forking the Doctrine: Customizing the Adaptive Manager

> **Level:** Advanced · **Premium** · **Prerequisites:** [Full ML Pipeline](./10-esteira-ml-completa.en.md), [Regime + Model](./09-regime-modelo.en.md), [Survival Test](./04-teste-sobrevivencia.en.md)

In examples 06 through 10, you built ML models, detected regime, measured microstructure, and assembled complete pipelines. But in all of them, execution was **raw**: the model's prediction becomes directly `comprado(1.0)` or `vendido(1.0)`. There's no risk manager, no stop, no trailing, no pyramid. The model says direction, and the backtest buys/sells at market with no position management.

The `grupo` library is CT Lab's adaptive manager — an order engine that tracks position, manages stops and takes, does trailing, and re-arms cycles. "Forking the doctrine" means customizing this manager: combining your model's prediction with the group's execution, creating your own trading doctrine.

> **Doctrine** is the set of rules that defines how you trade: when to enter, how much to enter, where to stop, when to exit. The model predicts direction; the doctrine executes. Without doctrine, the model is just an opinion.

---

## The problem

The survival test (example 04) showed that the random-entry floor in BTC is **EV = −0.92 réguas/trade with fees** — buying or selling at random moments loses money. The GBDT (example 06) overcame this floor with +$12,447 net PnL, but execution was raw — no stops, no trailing, no management.

The question is: **does the adaptive manager improve the model's result?**

To answer, you'll compare:
1. **GBDT raw** — prediction becomes direct buy/sell, no manager.
2. **GBDT + group** — prediction arms the group with stop and take, the manager handles exit.
3. **RSI reversal raw** — reversal signal in sideways, no manager.
4. **RSI reversal + group** — same strategy, with adaptive manager.

---

## Step 1 — The survival floor

Before any backtest, remember the floor. The survival test measures the EV of random-direction entry:

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 20,
    "fee_pct": 0.001
  }
}
```

### Result

| Metric | Value |
|---|---|
| Moments | 20 |
| EV (réguas/trade) | −0.922 |
| Sum PnL | −$24,609 |
| Positive pairs | 1/20 (5%) |

> The floor confirms: without entry criteria, the adaptive manager loses with fees. Each random trade costs 0.92 réguas of edge. **Your strategy needs to add +0.92 réguas of edge per trade just to break even.**

---

## Step 2 — The `grupo` library

The `grupo` library is imported at the top of the Rhai script:

```rhai
import "grupo" as g;
```

### Group anatomy

A group is a set of orders with **entradas** (entries that accumulate position) and **saídas** (exits: reduce-only stop/take/trailing). When position reaches zero through an exit, the group resolves and, if cycles remain, re-arms.

```rhai
let cfg = #{
  entradas: [
    #{ tipo: "market", lado: "compra", lote: 1.0 },
    #{ offset: -5.0, tipo: "limit", lado: "compra", lote: 2.0, validade: 10 }
  ],
  saidas: [
    #{ offset: -20.0, tipo: "stop", lote: 3.0 },
    #{ offset:  15.0, tipo: "limit", lote: 3.0 },
    #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 3.0 }
  ],
  ciclos: 1.0
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;
r.decisao
```

### Contract

| Parameter | Description |
|---|---|
| `cfg.entradas` | List of entry orders: `market`, `limit`, or `stop` |
| `cfg.saidas` | List of exit orders: `stop`, `limit`, or `trailing` |
| `cfg.ciclos` | Number of re-arms after resolving. `1.0` = arm, execute, resolve, stop. `0.0` = infinite |
| `cfg.referencia` | Reference price for offsets (default = `close`) |
| `estado` | Group state (passed by reference, always reassign) |
| `posicao` | Current position (passed by backtest) |
| `ordens` | Current bar orders (passed by backtest) |

> **The return `r.decisao`** is the bar's decision: can be `comprado(1.0)`, `vendido(1.0)`, `zerado()`, or `decisao(...)` with OCO orders. You must return this from the strategy.

---

## Step 3 — GBDT raw (baseline)

The GBDT from example 06, no manager — prediction becomes direct market order:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r11_gbdt3_pred",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r11_cru_fee"
  }
}
```

### Result

| Metric | With fee 0.1% | No fee |
|---|---|---|
| **PnL** | −$23,974 | +$76,991 |
| **Trades** | 785 | 785 |
| **Win rate** | 25.7% | 89.4% |
| **Profit factor** | 0.56 | 17.04 |
| **Sharpe** | 0.019 | 0.482 |
| **Fees** | $100,837 | 0 |
| **Exposure** | 99.7% | 99.7% |

> Without fees, raw GBDT has an 89.4% win rate and profit factor of 17 — seems extraordinary. But 785 trades: $100k in fees destroy everything. The model predicts direction almost every bar and the backtest executes nearly every candle.

---

## Step 4 — GBDT + group (fork)

Now, the prediction arms the group instead of direct buy/sell. The group enters at market, places stop at −2% and take at +3%, and manages exit:

```rhai
import "grupo" as g;

let dir = 0.0;
if ind["pred"][0] > 0.0 { dir = 1.0; }
if ind["pred"][0] < 0.0 { dir = -1.0; }

let sem_pos = abs(posicao) <= 1e-9;

if sem_pos && abs(dir) <= 1e-9 {
  zerado()
} else if sem_pos && abs(dir) > 1e-9 {
  // Arm group: market entry + stop 2% + take 3%
  let lado = if dir > 0.0 { "compra" } else { "venda" };
  let s = if dir > 0.0 { -2.0 } else { 2.0 };
  let t = if dir > 0.0 { 3.0 } else { -3.0 };
  let cfg = #{
    entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
    saidas: [
      #{ offset: s, tipo: "stop", lote: 1.0 },
      #{ offset: t, tipo: "limit", lote: 1.0 }
    ],
    ciclos: 1.0
  };
  let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
  estado = r.estado;
  r.decisao
} else {
  // Has position: exit if prediction flipped against
  let contra = (posicao > 1e-9 && ind["pred"][0] < 0.0)
            || (posicao < -1e-9 && ind["pred"][0] > 0.0);
  if contra { zerado() } else {
    // Re-pass group to keep stops active
    let lado = if posicao > 1e-9 { "compra" } else { "venda" };
    let s = if posicao > 1e-9 { -2.0 } else { 2.0 };
    let t = if posicao > 1e-9 { 3.0 } else { -3.0 };
    let cfg = #{
      entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
      saidas: [
        #{ offset: s, tipo: "stop", lote: 1.0 },
        #{ offset: t, tipo: "limit", lote: 1.0 }
      ],
      ciclos: 1.0
    };
    let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
    estado = r.estado;
    r.decisao
  }
}
```

### Result

| Metric | GBDT + group w/ fee | GBDT + group no fee |
|---|---|---|
| **PnL** | −$130 | −$2 |
| **Trades** | 1 | 1 |
| **Exposure** | 0.06% | 0.06% |
| **Avg duration** | 900 bars | 900 bars |

> **Only 1 trade!** The group armed a single position, the stop or take triggered after 900 bars (~9 days), and the group never re-arms because `ciclos: 1.0` means "arm once, execute, resolve, stop."

### The `ciclos: 1.0` problem

The `grupo` lib with `ciclos: 1.0` arms the group a single time. When position reaches zero (stop or take), the group resolves and **does not re-arm** — even if the prediction indicates a new direction. The result is that over 1712 candles, the manager only trades once.

> **For the manager to cycle continuously, use `ciclos: 0.0`** (infinite). Or, implement manual re-arming: when `sem_pos && abs(dir) > 0`, create a new group. The difference between `ciclos: 1` and `ciclos: 0` is the difference between "trade once" and "trade forever."

---

## Step 5 — RSI reversal: raw vs group

### RSI reversal raw (example 09)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r11_rsi_adx",
    "estrategia_script": "if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r11_rsi_cru_fee"
  }
}
```

### RSI reversal + group

```rhai
import "grupo" as g;

let dir = 0.0;
if ind["adx"][0] < 20.0 && ind["rsi"][0] < 30.0 { dir = 1.0; }
if ind["adx"][0] < 20.0 && ind["rsi"][0] > 70.0 { dir = -1.0; }

let sem_pos = abs(posicao) <= 1e-9;

if sem_pos && abs(dir) <= 1e-9 {
  zerado()
} else if sem_pos && abs(dir) > 1e-9 {
  let lado = if dir > 0.0 { "compra" } else { "venda" };
  let s = if dir > 0.0 { -1.5 } else { 1.5 };
  let t = if dir > 0.0 { 2.0 } else { -2.0 };
  let cfg = #{
    entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
    saidas: [
      #{ offset: s, tipo: "stop", lote: 1.0 },
      #{ offset: t, tipo: "limit", lote: 1.0 }
    ],
    ciclos: 1.0
  };
  let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
  estado = r.estado;
  r.decisao
} else {
  zerado()
}
```

### Comparison

| Metric | RSI raw w/ fee | RSI + group w/ fee | RSI raw no fee | RSI + group no fee |
|---|---|---|---|---|
| **PnL** | −$4,707 | −$109 | +$1,334 | +$19 |
| **Trades** | 47 | 1 | 47 | 1 |
| **Win rate** | 8.5% | 0% | 76.6% | 100% |
| **Profit factor** | 0.17 | 0 | 1.66 | — |
| **Exposure** | 4.8% | 0.06% | 4.8% | 0.06% |

> Same pattern: the group with `ciclos: 1.0` only trades once. Without a re-arm wrapper, the manager doesn't cycle.

---

## Step 6 — The discovery: you need the fork

The backtests reveal the central problem: **the `grupo` lib as default is not a complete strategy — it's an order engine.** It stores and manages orders, but doesn't decide when to re-arm. `ciclos: 1.0` is a conservative default: arm once, execute, resolve, stop.

To make the manager cycle, you need a **wrapper** — logic that detects when the group has resolved (position zeroed after having one) and arms a new group. This is the **doctrine fork**: you customize the manager's behavior.

### Fork: automatic re-arm with `ciclos: 0.0`

The simplest change is to use `ciclos: 0.0` (infinite). The group re-arms automatically after resolving:

```rhai
let cfg = #{
  entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
  saidas: [
    #{ offset: s, tipo: "stop", lote: 1.0 },
    #{ offset: t, tipo: "limit", lote: 1.0 }
  ],
  ciclos: 0.0  // ← infinite: re-arms after resolving
};
```

> **Warning**: with `ciclos: 0.0`, the group re-arms **in the same direction** as the previous group. If you want the new entry to follow the model's current prediction, you need `saidas_vivas: true` or manual re-centering.

### Fork: customize the library

The `grupo` lib is forkable by design. You can:

```json
{
  "name": "salvar_lib",
  "arguments": {
    "nome": "meu_grupo",
    "fonte": "// your fork of the grupo lib, with custom rules..."
  }
}
```

And import in the strategy:

```rhai
import "meu_grupo" as g;
```

Fork examples:

1. **Directional re-arm**: after resolving, the group re-arms in the direction of the current prediction (not the original direction).
2. **Cooldown re-arm**: after resolving, waits N bars before re-arming.
3. **Regime-gated re-arm**: only re-arms if ADX < 20 (sideways) or ADX > 25 (trending).
4. **Pyramid**: instead of a fixed lot, accumulates position at different price levels.
5. **ATR trailing**: trailing distance adapts to volatility (ATR).

> **The doctrine is YOURS.** The `grupo` lib is the seed — the order engine. The fork is where you define your re-arm rules, entry conditions, and exit parameters. The model predicts direction; the doctrine executes. The quality of the doctrine determines whether the model's edge survives execution costs.

---

## Step 7 — Saving and sharing your fork

### Save a library

```json
{
  "name": "salvar_lib",
  "arguments": {
    "nome": "minha_doutrina",
    "fonte": "// fork of grupo lib with directional re-arm and ATR trailing..."
  }
}
```

### List libraries

```json
{ "name": "listar_libs", "arguments": {} }
```

### Read a library

```json
{ "name": "ler_lib", "arguments": { "nome": "minha_doutrina" } }
```

### Use in backtest

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/meu_modelo_pred",
    "estrategia_script": "import \"minha_doutrina\" as g; // ... use g::tick ...",
    "nome": "backtest_doutrina"
  }
}
```

---

## Comparative table (all results)

| Strategy | Trades | PnL w/ fee | PnL no fee | Win% no fee | PF no fee |
|---|---|---|---|---|---|
| Survival (floor) | 40 | −$24,609 | — | — | — |
| GBDT raw | 785 | −$23,974 | +$76,991 | 89.4% | 17.0 |
| GBDT + group (1 cycle) | 1 | −$130 | −$2 | 0% | 0 |
| RSI reversal raw | 47 | −$4,707 | +$1,334 | 76.6% | 1.66 |
| RSI + group (1 cycle) | 1 | −$109 | +$19 | 100% | — |
| Buy & Hold | 0 | −$296 | — | — | — |

### Lessons

1. **Without fees, raw GBDT seems extraordinary** (PF 17, win 89%) — but 785 trades with $100k in fees. Overtrading is enemy #1.

2. **The manager with `ciclos: 1.0` is too conservative** — only trades once in 1712 candles. To trade continuously, needs `ciclos: 0.0` or a re-arm wrapper.

3. **Raw RSI reversal has 47 trades with PF 1.66 without fees** — the regime filter (ADX < 20) reduces turnover naturally, without needing the manager.

4. **The manager's value isn't in reducing trades — it's in managing risk.** Stop and trailing protect against catastrophic losses. Raw GBDT has max drawdown of 222%; the manager has 1.3%.

> **The doctrine doesn't replace the model — it protects it.** The model says direction; the doctrine defines where to stop if wrong. Without a stop, a single bad trade can decimate the capital. With a stop, losses are limited and the model can keep trading.

---

## Next steps

- **Implement `ciclos: 0.0`**: modify the scripts in this example to use `ciclos: 0.0` and measure the result. The manager should re-arm and generate more trades.
- **Fork with directional re-arm**: save a `meu_grupo` library that re-arms in the direction of the model's current prediction, not the group's original direction.
- **Fork with ATR trailing**: trailing distance adapts to ATR — high volatility = wider trailing, low volatility = tighter trailing.
- **Combine regime + doctrine**: only arm the group when ADX < 20 (sideways) and use RSI reversal as trigger. Regime filters when to trade; doctrine manages how to trade.

---

> Back to: [README](../README.md) · [Full ML Pipeline](./10-esteira-ml-completa.en.md) · [Regime + Model](./09-regime-modelo.en.md) · [Survival Test](./04-teste-sobrevivencia.en.md)

_Last updated: 2026-08-12_
