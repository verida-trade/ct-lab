# Rhai Strategy (Advanced)

> Full Rhai scripts with state, entry/exit logic, and indicator usage by alias.

For complex strategies that don't fit in one line, use a full Rhai script. The script can have state (variables persisting across bars), conditional logic, and call the `grupo` lib.

---

## State across bars

Variables declared with `let` persist across bars:

```rhai
let entry = 0.0;
let stop = 0.0;

if posicao == 0.0 && ind["rsi"][0] < 30.0 {
    entry = close[0];
    stop = close[0] * 0.98;
    comprado(1.0)
} else if posicao > 0.0 {
    if close[0] < stop {
        zerado()
    } else {
        let new_stop = close[0] * 0.98;
        if new_stop > stop { stop = new_stop; }
        comprado(1.0)
    }
} else {
    zerado()
}
```

---

## Using the `grupo` lib

For strategies with OCO stops, trailing, takes and cycles, import the `grupo` lib:

```rhai
import "grupo" as g;

let cfg = #{
    entradas: [
        #{ offset: 0.0, tipo: "limit", lado: "compra", lote: 1.0 },
        #{ offset: -5.0, tipo: "limit", lado: "compra", lote: 2.0, validade: 10 },
    ],
    saidas: [
        #{ offset: -20.0, tipo: "stop", lote: 3.0 },
        #{ offset: 15.0, tipo: "limit", lote: 3.0 },
        #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 3.0 },
    ],
    ciclos: 1.0,
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;
r.decisao
```

> See [The `grupo` lib](./05-lib-grupo.en.md) for full API details.

---

## Indicators by alias

Access indicators via `ind["alias"][0]` (current value) or `ind["alias"][1]` (previous bar):

```rhai
if ind["rsi"][0] < 30.0 && ind["adx"][0] > 25.0 {
    comprado(1.0)
} else if ind["rsi"][0] > 70.0 {
    vendido(1.0)
} else {
    zerado()
}
```

Aliases are defined in `indicadores_receitas` or come from the derived series in `indicadores`.

---

> Next: [Python strategy](./04-estrategia-python.en.md) · [The `grupo` lib](./05-lib-grupo.en.md)
