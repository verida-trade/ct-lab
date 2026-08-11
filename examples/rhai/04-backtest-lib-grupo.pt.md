# Receita 04 — Backtest com a lib `grupo`

> **Nível:** Intermediário · **Premium** · **Pré-requisitos:** [Lib grupo](../docs/05-backtest/05-lib-grupo.pt.md)

Estratégia usando a lib `grupo` com stop, take e trailing em OCO.

## Script Rhai

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

> **Nota:** Lembre de reatribuir `estado = r.estado` — sem isso o grupo re-arma toda barra.
