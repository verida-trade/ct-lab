# A Lib `grupo` (Seed CT)

> **Premium.** Grupos de ordens com entradas acumulativas e saídas OCO (stop/take/trailing). A fundação de execução do método CT.

A lib `grupo` é o motor de execução do seed CT. Um grupo = N entradas que acumulam posição (preço médio) + M saídas reduce-only em OCO. Quando a posição zera por saída, o grupo resolve e rearma se há ciclos.

---

## Estrutura

```rhai
import "grupo" as g;

let cfg = #{
    entradas: [...],   // acumulam (sem OCO entre si)
    saidas: [...],     // reduce-only, OCO entre si
    ciclos: 1.0,       // 1 = one-shot; K = rearma K vezes; 0 = contínuo
    // referencia: 100.0,  // preço de referência (default = close no arme)
    // prazo: 20,          // fecha a mercado após N barras
    // recentrar: true,   // re-arma contínuo enquanto nada executou
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;  // OBRIGATÓRIO: sem isso o grupo re-arma toda barra
r.decisao
```

---

## Entradas

Cada entrada acumula posição. Níveis são offsets da referência:

```rhai
entradas: [
    #{ offset: 0.0,  tipo: "limit", lado: "compra", lote: 1.0 },
    #{ offset: -5.0, tipo: "limit", lado: "compra", lote: 2.0, validade: 10 },
    #{ offset: 10.0, tipo: "stop",  lado: "compra", lote: 2.0 },
    #{ tipo: "market", lado: "compra", lote: 1.0 },
],
```

| Campo | Descrição |
|---|---|
| `offset` | Distância da referência (preço absoluto) |
| `tipo` | `"limit"`, `"stop"`, `"market"` |
| `lado` | `"compra"` ou `"venda"` |
| `lote` | Tamanho da ordem |
| `validade` | Cancela após N barras sem execução (opcional) |

---

## Saídas (OCO)

Saídas são reduce-only e compartilham um grupo OCO. Num candle que toca mais de uma, só a **pessimista** executa:

```rhai
saidas: [
    #{ offset: -20.0, tipo: "stop",  lote: 6.0 },
    #{ offset: 15.0,  tipo: "limit", lote: 3.0 },
    #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 3.0 },
],
```

| Tipo | Descrição |
|---|---|
| `"stop"` | Stop-loss a `offset` da referência |
| `"limit"` | Take-profit a `offset` da referência |
| `"trailing"` | Stop móvel a `distancia` do extremo favorável desde `ativacao` |

---

## Ciclos

| `ciclos` | Comportamento |
|---|---|
| `1.0` | One-shot — executa uma vez e para |
| `K` | Rearma K vezes após zerar |
| `0.0` | Contínuo — rearma indefinidamente |

---

## `validade` e `prazo`

- **`validade`** (por entrada): a ordem em descanso é cancelada se não executar em N barras — **não gera P&L** (nunca virou posição).
- **`prazo`** (na cfg): a posição aberta é **fechada a mercado** após N barras — saída explícita, realiza P&L.

---

## Estado

**Sempre reatribua o estado:**

```rhai
let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;  // SEM isso, o grupo re-arma toda barra (bug!)
r.decisao
```

---

## Salvar e usar a lib

A lib seed `grupo` está disponível em `ct://libs/seed/grupo`. Para forkar:

```json
{
  "name": "salvar_lib",
  "arguments": {
    "name": "meu_grupo",
    "content": "... seu Rhai customizado ..."
  }
}
```

Depois use `import "meu_grupo" as g;` na estratégia.

> Veja [Fork da lib `grupo`](./06-fork-lib-grupo.pt.md)

---

> Próximo: [Fork da lib](./06-fork-lib-grupo.pt.md) · [Camada adaptativa](./07-camada-adaptativa.pt.md)
