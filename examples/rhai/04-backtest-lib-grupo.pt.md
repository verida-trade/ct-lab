# Receita 04 — Backtest com a lib `grupo`

> **Nível:** Intermediário · **Plano:** Premium
> **Pré-requisito:** [docs/05-backtest/05-lib-grupo](../../../docs/05-backtest/05-lib-grupo)

---

## A lib `grupo`

- **Entradas escalonadas:** define uma ou mais ordens de entrada com `offset`, `tipo` (limit/market) e `lado` (compra/venda), disparadas em sequência.
- **Saídas OCO:** cada saída (stop, limit, trailing) compete no mesmo ciclo — a primeira a disparar cancela as demais.
- **Trailing stop:** acompanha o preço a uma `distancia` fixa, ativando apenas quando o lucro atinge `ativacao` pontos.
- **Ciclos:** controla quantas vezes o grupo rearma após uma saída (`ciclos: 1.0` = rearma uma vez; `ciclos: 0` = contínuo).

---

## Passo 1 — Buscar série

```
ct_buscar_binance("BTCUSDT", "15m")
```

Série utilizada: `ct://series/binance/BTCUSDT/15m` (1724 candles).

---

## Passo 2 — A estratégia

```rhai
import "grupo" as g;

// Configuração do grupo: 1 entrada limit, stop -20, TP +15, trailing 8/10
let cfg = #{
    entradas: [
        #{ offset: 0.0, tipo: "limit", lado: "compra", lote: 1.0 },
    ],
    saidas: [
        #{ offset: -20.0, tipo: "stop", lote: 1.0 },
        #{ offset: 15.0, tipo: "limit", lote: 1.0 },
        #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 1.0 },
    ],
    ciclos: 1.0,  // rearma uma vez após a primeira saída
};

// grupo::tick processa o candle atual e devolve decisão + novo estado
let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;  // CRÍTICO: persistir o estado entre barras
r.decisao
```

---

## Passo 3 — Backtest

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

---

## Resultado real

| Métrica | Com fee (0,1%) | Sem fee |
|---|---|---|
| **Trades** | 1 | 1 |
| **PnL** | −$148,58 | −$20,58 |
| **PnL bruto** | −$20,58 | −$20,58 |
| **Fees** | $128,00 | — |
| **Sharpe** | −0,024 | −0,024 |
| **Win rate** | 0% | 0% |
| **Profit factor** | 0 | 0 |
| **Max drawdown** | 14,86% | 2,06% |
| **Expectancy** | 0,058% | — |
| **Loss médio** | −$148,58 | — |
| **Long** | 1 | 1 |

---

## Por que só 1 trade?

- **`ciclos: 1.0`** permite apenas um rearme após a primeira saída. Como o stop foi atingido no primeiro ciclo, o grupo rearma uma vez — mas ocorre apenas 1 trade no período.
- **Stop muito justo:** −$20 em BTC a ~$64k representa apenas **0,03%** do preço. Em um candle de 15m, BTC oscila facilmente $50–$100, então o stop foi atingido quase imediatamente.
- **Trailing nunca ativou:** o trailing exigia que o preço subisse +$10 (ativacao) antes de começar a rastrear. Como o stop disparou primeiro, o trailing nunca chegou a ser avaliado.
- **Impacto das fees:** com `lote: 1.0` (1 BTC inteiro) e capital de $1000, a taxa de 0,1% por lado sobre ~$64k gera **$128** em fees — 6× maiores que a perda do stop de $20.

---

## ⚠️ Nota

A linha `estado = r.estado` é **obrigatória**. Sem ela, a variável `estado` nunca é atualizada entre barras e o `grupo` reinterpreta cada candle como se fosse o primeiro — rearmando ordens, perdendo a posição e gerando trades fantasmas. O `estado` carrega o ciclo atual, ordens pendentes e flags internas do grupo.

---

## Variações

- **Stop mais largo (−50):** troque `offset: -20.0` por `offset: -50.0` para dar mais respiro ao trade e evitar saída prematura.
- **Execução contínua:** defina `ciclos: 0` para que o grupo rearme indefinidamente após cada saída.
- **Entrada escalonada:** adicione `#{ offset: -10.0, tipo: "limit", lado: "compra", lote: 1.0 }` ao array `entradas` para abrir posição em dois níveis.
- **Aumentar lote:** mude `lote: 1.0` para `lote: 2.0` e observe como fees e PnL escalam proporcionalmente.
