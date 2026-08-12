# 03 — Backtest with the `grupo` Lib

> **Level:** Intermediate · **Premium** · **Prerequisites:** [The `grupo` Lib](../docs/05-backtest/05-lib-grupo.en.md), [Rhai Strategy](../docs/05-backtest/03-estrategia-rhai.en.md)

The `grupo` lib is CT Lab's execution engine. It lets you define **order groups** — multiple entries that accumulate position, OCO exits (stop / take / trailing), and automatic rearm cycles — all declaratively, in a single config block.

In this example, you will:

1. **Run a backtest** with staggered entries + stop, take-profit, and trailing in OCO.
2. **Understand the output** — metrics, trades, and what each field means.
3. **Vary parameters** with `ct_comparar` to see how different configs affect results.
4. **Fork the lib** to create your own version with custom rules.

---

## The problem

You want to test a **staggered accumulation** strategy:

- **Entry 1**: market buy immediately (0.1 BTC).
- **Entry 2**: limit at $200 below the reference price (0.1 BTC) — "buy the dip".
- **OCO exits**:
  - Stop-loss at $500 below (close everything).
  - Take-profit at $300 above (partial exit).
  - Trailing stop at $150 from the peak, activating after $100 in favor.
- **Continuous cycles**: after flattening, rearm immediately.

Doing this manually with `comprado()` / `zerado()` would require dozens of state lines. With the `grupo` lib, it's 12 lines.

---

## Step 1 — Fetch data

```
Fetch BTCUSDT at 15m from Binance.
```

```json
{
  "name": "buscar_serie",
  "arguments": {
    "provider": "binance",
    "symbol": "BTCUSDT",
    "interval": "15m"
  }
}
```

> The series is available at `ct://series/binance/BTCUSDT/15m`.

---

## Step 2 — Run the backtest

### The strategy (Rhai script)

```rhai
import "grupo" as g;

let cfg = #{
    entradas: [
        // Entry 1: immediate market — 0.1 BTC
        #{ offset: 0.0, tipo: "market", lado: "compra", lote: 0.1 },

        // Entry 2: limit at $200 below — "buy the dip"
        #{ offset: -200.0, tipo: "limit", lado: "compra", lote: 0.1, validade: 20 },
    ],
    saidas: [
        // Stop-loss: $500 below, closes entire position (0.2 BTC)
        #{ offset: -500.0, tipo: "stop", lote: 0.2 },

        // Take-profit: $300 above, exits half (0.1 BTC)
        #{ offset: 300.0, tipo: "limit", lote: 0.1 },

        // Trailing: $150 from peak, activates after $100 in favor, exits half (0.1 BTC)
        #{ tipo: "trailing", distancia: 150.0, ativacao: 100.0, lote: 0.1 },
    ],
    ciclos: 0.0,  // 0 = continuous (rearms after flattening)
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;
r.decisao
```

### The backtest call

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "import \"grupo\" as g;\nlet cfg = #{entradas:[#{offset:0.0,tipo:\"market\",lado:\"compra\",lote:0.1},#{offset:-200.0,tipo:\"limit\",lado:\"compra\",lote:0.1,validade:20}],saidas:[#{offset:-500.0,tipo:\"stop\",lote:0.2},#{offset:300.0,tipo:\"limit\",lote:0.1},#{tipo:\"trailing\",distancia:150.0,ativacao:100.0,lote:0.1}],ciclos:0.0};\nlet r=g::tick(cfg,estado,posicao,open[0],high[0],low[0],close[0],ordens);\nestado=r.estado;\nr.decisao",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "grupo_escalado"
  }
}
```

### Result (example with 1000 candles of BTCUSDT 15m)

```json
{
  "uri": "ct://backtest/grupo_escalado",
  "num_trades": 138,
  "pnl_total": -2565.25,
  "pnl_bruto": -161.15,
  "fees_totais": 2399.65,
  "retorno_total": -0.2565,
  "sharpe": -0.115,
  "win_rate": 0.370,
  "profit_factor": 0.323,
  "drawdown_max": 0.266,
  "num_wins": 51,
  "num_losses": 87,
  "avg_win": 23.99,
  "avg_loss": -43.50,
  "best_trade": 79.93,
  "worst_trade": -106.15,
  "max_wins_seguidos": 3,
  "max_perdas_seguidas": 8,
  "exposicao": 0.919
}
```

---

## Step 3 — Interpreting the results

| Metric | Value | What it means |
|---|---|---|
| `num_trades` | 138 | The strategy executed 138 buy+sell cycles |
| `pnl_bruto` | −$161 | Without fees, the strategy nearly breaks even — the "cost" of arbitrary side |
| `fees_totais` | $2,400 | 138 trades × ~$17/trade in fees (0.1% on ~$6.5k notional per partial trade) |
| `pnl_total` | −$2,565 | Gross PnL + fees — the final result |
| `win_rate` | 37% | 51 wins vs 87 losses — the strategy loses more often than it wins |
| `avg_win` | $24 | Average gain per winning trade |
| `avg_loss` | −$44 | Average loss per losing trade |
| `exposicao` | 92% | Time in market — continuous cycles = almost always positioned |

### The insight

The gross PnL is nearly zero (−$161) — this is the **survival floor**: an arbitrary-side strategy (buying and selling without entry criteria) doesn't bleed without fees. The real loss comes from **fees**: $2.4k across 138 trades. This is the core teaching of the CT doctrine:

> Execution cannot depend on getting the direction right. The cost of execution (fees + slippage) is the enemy. Reduce turnover or find edge that overcomes the cost.

---

## Step 4 — Varying parameters with `ct_comparar`

To see how different configurations affect results, use `ct_comparar` — it runs the base backtest + variants changing **one factor at a time**:

```json
{
  "name": "ct_comparar",
  "arguments": {
    "base": {
      "serie": "ct://series/binance/BTCUSDT/15m",
      "estrategia_script": "import \"grupo\" as g;\nlet cfg = #{entradas:[#{offset:0.0,tipo:\"market\",lado:\"compra\",lote:0.1},#{offset:-200.0,tipo:\"limit\",lado:\"compra\",lote:0.1,validade:20}],saidas:[#{offset:-500.0,tipo:\"stop\",lote:0.2},#{offset:300.0,tipo:\"limit\",lote:0.1},#{tipo:\"trailing\",distancia:150.0,ativacao:100.0,lote:0.1}],ciclos:0.0};\nlet r=g::tick(cfg,estado,posicao,open[0],high[0],low[0],close[0],ordens);\nestado=r.estado;\nr.decisao",
      "capital_inicial": 10000,
      "fee_pct": 0.001
    },
    "variantes": [
      {
        "nome": "wider_stop",
        "parametros": {}
      },
      {
        "nome": "no_fee",
        "fee_pct": 0.0
      },
      {
        "nome": "higher_fee",
        "fee_pct": 0.002
      }
    ]
  }
}
```

The `no_fee` variant isolates the effect of fees: you'll see that gross PnL is nearly zero, confirming that a strategy without entry criteria has no edge — only costs.

---

## Strategy anatomy

> 💡 The `grupo` lib comes pre-installed with CT Lab Desktop (Premium license). No manual installation needed — `import "grupo" as g` works out-of-the-box. To create your own version, use `salvar_lib` with a fork of the seed (`ct://libs/seed/grupo`).

```
                    ┌─────────────────────────────────────┐
                    │           Configuration (cfg)        │
                    ├─────────────────────────────────────┤
                    │                                     │
                    │  ENTRIES (accumulate, no OCO)       │
                    │  ┌───────────────────────────┐      │
                    │  │ 1. market @ current price │ 0.1  │
                    │  │ 2. limit @ −$200 (expires  │ 0.1  │
                    │  │    after 20 bars)          │      │
                    │  └───────────────────────────┘      │
                    │                  │                   │
                    │                  ▼                    │
                    │  ACCUMULATED POSITION (up to 0.2 BTC)│
                    │                  │                   │
                    │                  ▼                    │
                    │  EXITS (reduce-only, OCO)            │
                    │  ┌───────────────────────────┐      │
                    │  │ A. stop @ −$500    │ 0.2   │      │
                    │  │ B. limit @ +$300   │ 0.1   │      │
                    │  │ C. trailing $150   │ 0.1   │      │
                    │  │    (activates +$100)       │      │
                    │  └───────────────────────────┘      │
                    │                  │                   │
                    │   OCO: only one fires per candle     │
                    │   (the pessimistic one on ties)       │
                    │                  │                   │
                    │                  ▼                    │
                    │  Position zeroed? → rearm (ciclos: 0) │
                    └─────────────────────────────────────┘
```

### Config fields

| Field | Type | Description |
|---|---|---|
| `entradas[].offset` | f64 | Distance from reference (absolute price, not %) |
| `entradas[].tipo` | `"market"` \| `"limit"` \| `"stop"` | Order type |
| `entradas[].lado` | `"compra"` \| `"venda"` | Side |
| `entradas[].lote` | f64 | Order size |
| `entradas[].validade` | f64? | Cancels after N bars if unfilled |
| `saidas[].offset` | f64 | Distance from reference (stop/limit) |
| `saidas[].tipo` | `"stop"` \| `"limit"` \| `"trailing"` | Exit type |
| `saidas[].lote` | f64 | Exit size (reduce-only) |
| `saidas[].distancia` | f64 | Distance from favorable extreme (trailing) |
| `saidas[].ativacao` | f64? | Offset in favor to activate trailing |
| `ciclos` | f64 | 1 = one-shot; K = rearm K times; 0 = continuous |
| `prazo` | f64? | Close at market after N bars |
| `referencia` | f64? | Reference price (default = close at arme) |
| `recentrar` | bool? | Re-arms continuous while nothing has filled |

> ⚠️ **Always reassign state**: `estado = r.estado;` — without this, the group rearms every bar, losing fill tracking.

---

## Next steps

- **Add entry criteria**: combine the `grupo` lib with indicators (RSI, ADX) to only arm when there's a signal — see [RSI + ADX Filter](./02-rsi-filtro-adx.en.md).
- **Fork the lib**: create your own version with custom rules — see [Forking the `grupo` Lib](../docs/05-backtest/06-fork-lib-grupo.en.md).
- **Survival test**: validate that your strategy survives without getting direction right — see [Survival Test](./04-teste-sobrevivencia.en.md).

---

> Back to: [README](../README.en.md) · [The `grupo` Lib](../docs/05-backtest/05-lib-grupo.en.md) · [Forking the Lib](../docs/05-backtest/06-fork-lib-grupo.en.md)

_Last updated: 2026-08-11_
