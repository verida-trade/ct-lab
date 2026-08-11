# Camada Adaptativa da Lib `grupo`

> **Premium.** Risco corrige a mercado; oportunidade espera em descanso. A camada adaptativa melhora a forma da distribuição — não cria EV.

A lib `grupo` suporta canais adaptativos que permitem à política re-decidir por barra, ajustando o risco a mercado.

---

## `alvo_pos` — ajuste de risco a mercado

Alvo absoluto de posição (assinado), idempotente. Só com posição aberta. Nunca inverte o lado (clamp a 0):

```rhai
let cfg = #{
    entradas: [...],
    saidas: [...],
    ciclos: 1.0,
    alvo_pos: 2.0,  // ajusta para 2.0 de posição (mantém ou aumenta)
};
```

- `alvo_pos: 0` fecha a posição já.
- `alvo_pos: 3.0` piramidar (aumenta de 1.0 para 3.0).
- `alvo_pos: 0.5` reduzir (diminui de 1.0 para 0.5).
- A histerese (ajustar por degraus) é responsabilidade da política.

---

## `saidas_vivas` — plano de saídas atualizável

```rhai
let cfg = #{
    entradas: [...],          // congeladas no arme
    saidas: [...],            // substituídas a cada barra quando saidas_vivas = true
    ciclos: 1.0,
    saidas_vivas: true,
};
```

As SAÍDAS da cfg corrente substituem as congeladas a cada barra (mover stop a breakeven, reescalar takes/trailing com a vol). Entradas permanecem congeladas no arme (rastreio de fills). O estado do trailing é preservado por índice.

---

## Regras adaptativas (no gestor)

| Regra | O que faz | Efeito |
|---|---|---|
| Breakeven | Move stop para o preço de entrada quando a posição favorece | Corta cauda esquerda |
| Reescala com vol | Ajusta stops/takes com a volatilidade local | Adapta ao regime |
| Pirâmide fora de vol alta | Adiciona posição quando vol baixa | Aumenta captura direita |

> **Importante:** A adaptação NÃO cria EV (teorema) — melhora a **forma** da distribuição: cauda esquerda −50%, captura direita +30-40%.

---

> Próximo: [Teste de sobrevivência](./08-teste-sobrevivencia.pt.md)
