# Recipe 04 — Backtest with `grupo` lib

> **Level:** Intermediate · **Premium** · **Prerequisites:** [grupo lib](../docs/05-backtest/05-lib-grupo.en.md)

Strategy using the `grupo` lib with stop, take and trailing in OCO.

## Rhai script

```rhai
import "grupo" as g;

let cfg = #{
    entradas: [
        #{ offset: 0.0, tipo: "limit", lado: "compra", lote: 1.0 },
    ],
    saidas: [
        #{ offset: -20.0, tipo: "stop", lote: 1.0 },
        #{ offset: 15.0, tipo: "limit", lote: 1.0 },
        #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 1.0 },
    ],
    ciclos: 1.0,
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;
r.decisao
```

## Backtest

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "import \"grupo\" as g;\nlet cfg = #{entradas:[#{offset:0.0,tipo:\"limit\",lado:\"compra\",lote:1.0}],saidas:[#{offset:-20.0,tipo:\"stop\",lote:1.0},#{offset:15.0,tipo:\"limit\",lote:1.0},#{tipo:\"trailing\",distancia:8.0,ativacao:10.0,lote:1.0}],ciclos:1.0};\nlet r=g::tick(cfg,estado,posicao,open[0],high[0],low[0],close[0],ordens);\nestado=r.estado;\nr.decisao",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "grupo_stop_trailing"
  }
}
```

> **Note:** Remember to reassign `estado = r.estado` — without it the group rearms every bar.
