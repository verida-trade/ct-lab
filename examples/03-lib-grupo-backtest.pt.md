# 03 — Backtest com a Lib `grupo`

> **Nível:** Intermediário · **Premium** · **Pré-requisitos:** [Lib `grupo`](../docs/05-backtest/05-lib-grupo.pt.md), [Estratégia Rhai](../docs/05-backtest/03-estrategia-rhai.pt.md)

A lib `grupo` é o motor de execução do CT Lab. Ela permite definir **conjuntos de ordens** — múltiplas entradas que acumulam posição, saídas em OCO (stop / take / trailing) e ciclos automáticos de rearme — tudo declarativo, em um único bloco de configuração.

Neste exemplo, você vai:

1. **Rodar um backtest** com entradas escalonadas + stop, take-profit e trailing em OCO.
2. **Entender a saída** — métricas, trades e o que cada campo significa.
3. **Variar parâmetros** com `ct_comparar` para ver como diferentes configurações afetam o resultado.
4. **Forkar a lib** para criar sua própria versão com regras customizadas.

---

## O problema

Você quer testar uma estratégia de **acumulação escalonada**:

- **Entrada 1**: compra a mercado imediatamente (0.1 BTC).
- **Entrada 2**: limit a $200 abaixo do preço de referência (0.1 BTC) — "comprar a queda".
- **Saídas em OCO**:
  - Stop-loss a $500 abaixo (fecha tudo).
  - Take-profit a $300 acima (realiza parcial).
  - Trailing stop a $150 do topo, ativando após $100 a favor.
- **Ciclos contínuos**: após zerar, rearme imediatamente.

Fazer isso manualmente com `comprado()` / `zerado()` exigiria dezenas de linhas de estado. Com a lib `grupo`, são 12 linhas.

---

## Passo 1 — Buscar dados

```
Busque BTCUSDT em 15m na Binance.
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

> A série fica disponível em `ct://series/binance/BTCUSDT/15m`.

---

## Passo 2 — Rodar o backtest

### A estratégia (script Rhai)

```rhai
import "grupo" as g;

let cfg = #{
    entradas: [
        // Entrada 1: market imediato — 0.1 BTC
        #{ offset: 0.0, tipo: "market", lado: "compra", lote: 0.1 },

        // Entrada 2: limit a $200 abaixo — "comprar a queda"
        #{ offset: -200.0, tipo: "limit", lado: "compra", lote: 0.1, validade: 20 },
    ],
    saidas: [
        // Stop-loss: $500 abaixo, fecha a posição inteira (0.2 BTC)
        #{ offset: -500.0, tipo: "stop", lote: 0.2 },

        // Take-profit: $300 acima, realiza metade (0.1 BTC)
        #{ offset: 300.0, tipo: "limit", lote: 0.1 },

        // Trailing: $150 do topo, ativa após $100 a favor, realiza metade (0.1 BTC)
        #{ tipo: "trailing", distancia: 150.0, ativacao: 100.0, lote: 0.1 },
    ],
    ciclos: 0.0,  // 0 = contínuo (rearma após zerar)
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;
r.decisao
```

### A chamada do backtest

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

### Resultado (exemplo com 1000 candles de BTCUSDT 15m)

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
  "melhor_trade": 79.93,
  "pior_trade": -106.15,
  "max_wins_seguidos": 3,
  "max_perdas_seguidas": 8,
  "exposicao": 0.919
}
```

---

## Passo 3 — Interpretando os resultados

| Métrica | Valor | O que significa |
|---|---|---|
| `num_trades` | 138 | A estratégia executou 138 ciclos de compra+venda |
| `pnl_bruto` | −$161 | Sem taxas, a estratégia quase empata — o "custo" do lado arbitrário |
| `fees_totais` | $2.400 | 138 trades × ~$17/trade de fee (0.1% sobre ~$6.5k de notional por trade parcial) |
| `pnl_total` | −$2.565 | PnL bruto + fees — o resultado final |
| `win_rate` | 37% | 51 ganhos vs 87 perdas — a estratégia perde mais do que ganha |
| `avg_win` | $24 | Ganho médio por trade vencedor |
| `avg_loss` | −$44 | Perda média por trade perdedor |
| `exposicao` | 92% | Tempo no mercado — ciclos contínuos = quase sempre posicionado |

### O insight

O PnL bruto é quase zero (−$161) — isso é o **piso de sobrevivência**: uma estratégia de lado arbitrário (compra e vende sem critério de entrada) não sangra sem taxas. A perda real vem dos **fees**: $2.4k em 138 trades. Este é o ensinamento central da doutrina CT:

> A execução não pode depender de acertar lado. O custo de execução (fees + slippage) é o inimigo. Reduza o turnover ou encontre edge que supere o custo.

---

## Passo 4 — Variando parâmetros com `ct_comparar`

Para ver como diferentes configurações afetam o resultado, use `ct_comparar` — ela roda o backtest base + variantes mudando **um fator por vez**:

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
        "nome": "stop_largo",
        "parametros": {}
      },
      {
        "nome": "sem_fee",
        "fee_pct": 0.0
      },
      {
        "nome": "fee_maior",
        "fee_pct": 0.002
      }
    ]
  }
}
```

A variante `sem_fee` isola o efeito das taxas: você verá que o PnL bruto é quase zero, confirmando que a estratégia sem critério de entrada não tem edge — apenas custos.

---

## Anatomia da estratégia

> 💡 A lib `grupo` já vem pré-instalada com o CT Lab Desktop (licença Premium). Não é preciso instalar manualmente — o `import "grupo" as g` funciona out-of-the-box. Para criar sua própria versão, use `salvar_lib` com um fork do seed (`ct://libs/seed/grupo`).

```
                    ┌─────────────────────────────────────┐
                    │           Configuração (cfg)          │
                    ├─────────────────────────────────────┤
                    │                                     │
                    │  ENTRADAS (acumulam, sem OCO)       │
                    │  ┌───────────────────────────┐      │
                    │  │ 1. market @ preço atual   │ 0.1  │
                    │  │ 2. limit @ −$200 (validade│ 0.1  │
                    │  │    20 barras)              │      │
                    │  └───────────────────────────┘      │
                    │                  │                   │
                    │                  ▼                    │
                    │  POSIÇÃO ACUMULADA (até 0.2 BTC)    │
                    │                  │                   │
                    │                  ▼                    │
                    │  SAÍDAS (reduce-only, OCO)          │
                    │  ┌───────────────────────────┐      │
                    │  │ A. stop @ −$500    │ 0.2   │      │
                    │  │ B. limit @ +$300   │ 0.1   │      │
                    │  │ C. trailing $150  │ 0.1   │      │
                    │  │    (ativa +$100)  │       │      │
                    │  └───────────────────────────┘      │
                    │                  │                   │
                    │   OCO: só uma executa por candle     │
                    │   (a pessimista em caso de empate)   │
                    │                  │                   │
                    │                  ▼                    │
                    │  Posição zerou? → rearma (ciclos: 0) │
                    └─────────────────────────────────────┘
```

### Campos da configuração

| Campo | Tipo | Descrição |
|---|---|---|
| `entradas[].offset` | f64 | Distância da referência (preço absoluto, não %) |
| `entradas[].tipo` | `"market"` \| `"limit"` \| `"stop"` | Tipo da ordem |
| `entradas[].lado` | `"compra"` \| `"venda"` | lado da entrada |
| `entradas[].lote` | f64 | Tamanho da ordem |
| `entradas[].validade` | f64? | Cancela após N barras sem execução |
| `saidas[].offset` | f64 | Distância da referência (stop/limit) |
| `saidas[].tipo` | `"stop"` \| `"limit"` \| `"trailing"` | Tipo da saída |
| `saidas[].lote` | f64 | Tamanho da saída (reduce-only) |
| `saidas[].distancia` | f64 | Distância do extremo favorável (trailing) |
| `saidas[].ativacao` | f64? | Offset a favor para ativar o trailing |
| `ciclos` | f64 | 1 = one-shot; K = rearma K vezes; 0 = contínuo |
| `prazo` | f64? | Fecha a mercado após N barras |
| `referencia` | f64? | Preço de referência (default = close no arme) |
| `recentrar` | bool? | Re-arma contínuo enquanto nada executou |

> ⚠️ **Sempre reatribua o estado**: `estado = r.estado;` — sem isso, o grupo re-arma toda barra, perdendo o rastreio de fills.

---

## Próximos passos

- **Adicionar critério de entrada**: combine a lib `grupo` com indicadores (RSI, ADX) para só armar quando houver sinal — veja [RSI + Filtro ADX](./02-rsi-filtro-adx.pt.md).
- **Forkar a lib**: crie sua própria versão com regras customizadas — veja [Fork da lib `grupo`](../docs/05-backtest/06-fork-lib-grupo.pt.md).
- **Teste de sobrevivência**: valide que sua estratégia sobrevive sem acertar lado — veja [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md).

---

> Voltar para: [README](../README.md) · [Lib `grupo`](../docs/05-backtest/05-lib-grupo.pt.md) · [Fork da lib](../docs/05-backtest/06-fork-lib-grupo.pt.md)

_Last updated: 2026-08-11_
