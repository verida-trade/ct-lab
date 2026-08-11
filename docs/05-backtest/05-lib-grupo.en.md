# The `grupo` Lib (CT Seed)

> **Premium.** Order groups with cumulative entries and OCO exits (stop/take/trailing). The execution foundation of the CT method.

The `grupo` lib is the execution engine of the CT seed. A group = N entries that accumulate position (average price) + M reduce-only exits in OCO. When the position zeroes via exit, the group resolves and rearms if there are cycles.

---

## Structure

```rhai
import "grupo" as g;

let cfg = #{
    entradas: [...],
    saidas: [...],
    ciclos: 1.0,
    // referencia: 100.0,
    // prazo: 20,
    // recentrar: true,
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;
r.decisao
```

---

## Entries

Each entry accumulates position. Levels are offsets from the reference:

```rhai
entradas: [
    #{ offset: 0.0,  tipo: "limit", lado: "compra", lote: 1.0 },
    #{ offset: -5.0, tipo: "limit", lado: "compra", lote: 2.0, validade: 10 },
    #{ offset: 10.0, tipo: "stop",  lado: "compra", lote: 2.0 },
    #{ tipo: "market", lado: "compra", lote: 1.0 },
],
```

| Field | Description |
|---|---|
| `offset` | Distance from reference |
| `tipo` | `"limit"`, `"stop"`, `"market"` |
| `lado` | `"compra"` (buy) or `"venda"` (sell) |
| `lote` | Order size |
| `validade` | Cancel after N bars without fill (optional) |

---

## Exits (OCO)

Exits are reduce-only and share an OCO group. In a candle that touches multiple, only the **pessimistic** one executes:

```rhai
saidas: [
    #{ offset: -20.0, tipo: "stop",  lote: 6.0 },
    #{ offset: 15.0,  tipo: "limit", lote: 3.0 },
    #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 3.0 },
],
```

| Type | Description |
|---|---|
| `"stop"` | Stop-loss at `offset` from reference |
| `"limit"` | Take-profit at `offset` from reference |
| `"trailing"` | Trailing stop at `distancia` from favorable extreme since `ativacao` |

---

## Cycles

| `ciclos` | Behavior |
|---|---|
| `1.0` | One-shot — execute once and stop |
| `K` | Rearm K times after zeroing |
| `0.0` | Continuous — rearm indefinitely |

---

## `validade` and `prazo`

- **`validade`** (per entry): resting order is cancelled if it doesn't fill in N bars — **no P&L** (never became position).
- **`prazo`** (in cfg): open position is **closed at market** after N bars — explicit exit, realizes P&L.

---

## State

**Always reassign state:**

```rhai
let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;  // WITHOUT this, the group rearms every bar (bug!)
r.decisao
```

---

## Saving and using the lib

The seed `grupo` lib is available at `ct://libs/seed/grupo`. To fork:

```json
{
  "name": "salvar_lib",
  "arguments": {
    "name": "my_grupo",
    "content": "... your custom Rhai ..."
  }
}
```

Then use `import "my_grupo" as g;` in your strategy.

> See [Forking the `grupo` lib](./06-fork-lib-grupo.en.md)

---

> Next: [Forking the lib](./06-fork-lib-grupo.en.md) · [Adaptive layer](./07-camada-adaptativa.en.md)
