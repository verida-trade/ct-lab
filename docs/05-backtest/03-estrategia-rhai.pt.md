# Estratégia em Rhai (Avançada)

> Scripts Rhai completos com estado, lógica de entrada/saída e uso de indicadores por alias.

Para estratégias mais complexas que não cabem em uma linha, use um script Rhai completo. O script pode ter estado (variáveis persistentes entre barras), lógica condicional e chamar a lib `grupo`.

---

## Estado entre barras

Variáveis declaradas com `let` persistem entre barras:

```rhai
// Estratégia com trailing stop
let entrada = 0.0;
let stop = 0.0;

if posicao == 0.0 && ind["rsi"][0] < 30.0 {
    entrada = close[0];
    stop = close[0] * 0.98;
    comprado(1.0)
} else if posicao > 0.0 {
    if close[0] < stop {
        zerado()
    } else {
        let novo_stop = close[0] * 0.98;
        if novo_stop > stop { stop = novo_stop; }
        comprado(1.0)
    }
} else {
    zerado()
}
```

---

## Usando a lib `grupo`

Para estratégias com stops OCO, trailing, takes e ciclos, importe a lib `grupo`:

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

> Veja [A lib `grupo`](./05-lib-grupo.pt.md) para detalhes completos da API.

---

## Indicadores por alias

Acesse indicadores via `ind["alias"][0]` (valor atual) ou `ind["alias"][1]` (barra anterior):

```rhai
if ind["rsi"][0] < 30.0 && ind["adx"][0] > 25.0 {
    comprado(1.0)
} else if ind["rsi"][0] > 70.0 {
    vendido(1.0)
} else {
    zerado()
}
```

Os aliases são definidos em `indicadores_receitas` ou vindos da série derivada em `indicadores`.

---

## Parâmetros expostos

Use `par["nome"]` para acessar parâmetros passados na chamada:

```json
{ "parametros": { "stop_pct": 0.02, "take_pct": 0.05 } }
```

```rhai
let stop = ind["entry_price"] * (1.0 - par["stop_pct"]);
let take = ind["entry_price"] * (1.0 + par["take_pct"]);
```

---

> Próximo: [Estratégia em Python](./04-estrategia-python.pt.md) · [A lib `grupo`](./05-lib-grupo.pt.md)
