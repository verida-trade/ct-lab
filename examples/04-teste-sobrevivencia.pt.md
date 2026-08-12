# 04 — Teste de Sobrevivência

> **Nível:** Intermediário · **Premium** · **Pré-requisitos:** [Backtest com a lib `grupo`](./03-lib-grupo-backtest.pt.md), [Lib `grupo`](../docs/05-backtest/05-lib-grupo.pt.md)

O teste de sobrevivência é o exame fundamental da doutrina CT Lab. Ele responde uma pergunta desconfortável:

> **Se você comprar e vender em momentos arbitrários — sem nenhum critério de entrada — quanto você perde?**

A tool `ct_testar_sobrevivencia` dispara **N momentos** espaçados no período. Cada momento abre **compra E venda** simultaneamente, usando o mesmo gestor adaptativo (fork da lib `grupo`). O resultado mede o **piso de lado arbitrário**: se o par (compra + venda) sobrevive sem taxas, sua estratégia tem sobrevida para competir. Se não sobrevive, nenhum fator de entrada vai salvar.

Neste exemplo, você vai:

1. **Rodar o teste de sobrevivência** com os parâmetros padrão (20 momentos).
2. **Interpretar o resultado** — o que significa cada métrica e como ler `por_momento`.
3. **Variar parâmetros** (stop, pirâmide, breakeven, reescala) para ver quais regras adaptativas pesam.
4. **Adicionar taxas** para ver o efeito devastador do custo de execução.
5. **Concluir** se sua estratégia tem ou não tem sobrevida.

---

## O problema

Você tem uma estratégia de acumulação escalonada com stop, take-profit e trailing. No [exemplo 03](./03-lib-grupo-backtest.pt.md) você viu que o backtest de 138 trades teve PnL bruto de −$161 (quase zero) — o piso de sobrevivência parecia existir.

Mas aquela era **uma só direção** (compra). No mercado real, você não sabe o lado. O teste de sobrevivência pergunta: e se você tivesse comprado **e** vendido no mesmo instante, com o mesmo gestor? O par só sobrevive se **a soma dos dois PnLs for ≥ 0**.

> A régua é a unidade de distância adaptativa: 1 régua = amplitude média da janela `w_vol` (64 barras por padrão). O stop de 0.5 régua se ajusta à volatilidade corrente — em períodos calmos, o stop é mais próximo; em períodos turbulentos, mais largo.

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

## Passo 2 — Rodar o teste (padrão: 20 momentos, sem taxas)

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m"
  }
}
```

Todos os parâmetros têm defaults da doutrina:
- `momentos`: 20 (o teste clássico)
- `stop_r`: 0.5 régua
- `lib_grupo`: `"grupo"` (a lib pré-instalada)
- `piramide`: true, `breakeven`: true, `reescala_vol`: true
- `fee_pct`: 0 (o teste é sem taxas — medimos o piso)
- `w_vol`: 64, `w_tercil`: 480

### Resultado (1000 candles de BTCUSDT 15m)

```json
{
  "n_momentos": 20,
  "soma_pnl": -3747.39,
  "soma_par_reguas": -1.719,
  "ev_par_reguas": -0.0859,
  "pares_positivos": 8,
  "por_momento": [
    { "ts": 1785447900, "regua": 1233, "pnl_compra":    0, "pnl_venda": -617, "par_reguas": -0.500 },
    { "ts": 1785484800, "regua": 1548, "pnl_compra": -774, "pnl_venda":  280, "par_reguas": -0.319 },
    { "ts": 1785522600, "regua": 2031, "pnl_compra": -329, "pnl_venda":  329, "par_reguas":  0.000 },
    { "ts": 1785559500, "regua": 1383, "pnl_compra": -692, "pnl_venda":    0, "par_reguas": -0.500 },
    { "ts": 1785597300, "regua":  289, "pnl_compra": -145, "pnl_venda":  803, "par_reguas":  2.275 },
    { "ts": 1785635100, "regua":  852, "pnl_compra":    0, "pnl_venda": -636, "par_reguas": -0.747 },
    { "ts": 1785672000, "regua": 1076, "pnl_compra":    0, "pnl_venda": -536, "par_reguas": -0.498 },
    { "ts": 1785709800, "regua":  814, "pnl_compra": -407, "pnl_venda":  660, "par_reguas":  0.311 },
    { "ts": 1785747600, "regua": 1496, "pnl_compra":  975, "pnl_venda": -748, "par_reguas":  0.151 },
    { "ts": 1785784500, "regua": 1760, "pnl_compra":  616, "pnl_venda": -890, "par_reguas": -0.156 },
    { "ts": 1785822300, "regua": 1108, "pnl_compra":  659, "pnl_venda": -537, "par_reguas":  0.110 },
    { "ts": 1785860100, "regua":  922, "pnl_compra":    0, "pnl_venda": -461, "par_reguas": -0.500 },
    { "ts": 1785897000, "regua":  934, "pnl_compra": -467, "pnl_venda":    0, "par_reguas": -0.500 },
    { "ts": 1785934800, "regua":  631, "pnl_compra":  451, "pnl_venda": -316, "par_reguas":  0.215 },
    { "ts": 1785972600, "regua": 1145, "pnl_compra":    0, "pnl_venda": -573, "par_reguas": -0.500 },
    { "ts": 1786009500, "regua":  586, "pnl_compra": -293, "pnl_venda":    0, "par_reguas": -0.500 },
    { "ts": 1786047300, "regua":  827, "pnl_compra":  540, "pnl_venda": -414, "par_reguas":  0.152 },
    { "ts": 1786085100, "regua":  821, "pnl_compra":  658, "pnl_venda": -364, "par_reguas":  0.358 },
    { "ts": 1786122000, "regua": 1225, "pnl_compra":  524, "pnl_venda": -612, "par_reguas": -0.073 },
    { "ts": 1786159800, "regua":  866, "pnl_compra":    0, "pnl_venda": -433, "par_reguas": -0.500 }
  ]
}
```

---

## Passo 3 — Interpretando os resultados

### Métricas de cabeçalho

| Métrica | Valor | O que significa |
|---|---|---|
| `n_momentos` | 20 | 20 momentos de disparo espaçados no período |
| `soma_pnl` | −$3.747 | Soma de todos os PnLs (compra + venda) — **negativo = o par sangra** |
| `soma_par_reguas` | −1.72 | Soma dos pares em réguas — negativa, mas pequena vs. 20 momentos |
| `ev_par_reguas` | −0.086 | Valor esperado por momento: −0.086 réguas por disparo |
| `pares_positivos` | 8 | 8 de 20 momentos (40%) tiveram par ≥ 0 |

### Lendo `por_momento`

Cada momento dispara **compra E venda** no mesmo instante. O `par_reguas` é a soma normalizada pela régua:

```
par_reguas = (pnl_compra + pnl_venda) / regua
```

- **`par_reguas = 0`** (momento 3): compra perdeu exatamente o que venda ganhou — empate perfeito.
- **`par_reguas = −0.5`** (momentos 1, 4, 12, 13, 15, 16, 20): um lado foi estopado e o outro não executou — o pior caso.
- **`par_reguas = +2.28`** (momento 5): a régua era muito curta (289), então o par capturou movimento desproporcional — o melhor momento.

### A leitura fundamental

O valor esperado é **−0.086 réguas por momento**. Em 20 momentos, isso acumula −1.72 réguas. O par (compra + venda) **não sobrevive** — ele sangra ~8.6% da régua por disparo, sem taxas.

> Isso **não é um fracasso da estratégia**. É o resultado esperado de lado arbitrário: sem critério de entrada, o gestor adaptativo não consegue compensar o custo do stop em ambos os lados. O teste diz: **sua estratégia precisa de um fator de entrada que adicione pelo menos +0.086 réguas por trade para chegar no zero**.

---

## Passo 4 — Variando parâmetros

Agora vamos ver quais regras adaptativas pesam mais. Cada variação muda um parâmetro, mantendo o resto no padrão:

### Tabela comparativa (20 momentos, sem taxas)

| Variante | `stop_r` | `piramide` | `breakeven` | `reescala_vol` | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|---|---|---|---|
| **Padrão** | 0.5 | ✓ | ✓ | ✓ | −$3.747 | −0.086 | 8/20 |
| Stop largo | 1.0 | ✓ | ✓ | ✓ | −$3.443 | −0.100 | 9/20 |
| **Stop curto** | **0.25** | ✓ | ✓ | ✓ | **−$2.733** | **−0.017** | **4/20** |
| Sem pirâmide | 0.5 | ✗ | ✓ | ✓ | −$2.437 | −0.071 | 9/20 |
| Sem breakeven | 0.5 | ✓ | ✗ | ✓ | −$6.675 | −0.237 | 8/20 |
| Sem reescala | 0.5 | ✓ | ✓ | ✗ | −$2.384 | −0.054 | 9/20 |

### Exemplo: stop mais curto (0.25 régua)

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "stop_r": 0.25
  }
}
```

### O que aprendemos

1. **Stop curto (0.25) é o melhor sobrevivente**: EV de só −0.017 réguas/momento. Cortar perdas rápido funciona — o stop é o que mais sangra em lado arbitrário. Mas atenção: pares_positivos cai para 4/20 — o stop curto limita ambos os lados, então poucos momentos capturam movimento grande.

2. **Breakeven é a regra que mais pesa**: sem ele, o EV piora de −0.086 para −0.237 (2.8× pior). O breakeven move o stop para o ponto de entrada após andar a favor, limitando o dano quando o mercado reverte.

3. **Pirâmide e reescala ajudam pouco**: desligá-las até melhora levemente o EV. Pirâmide aumenta exposição quando confirma, mas em lado arbitrário, confirmação é ruído.

4. **Stop largo (1.0) é pior que o padrão (0.5)**: mais régua deixada na mesa quando o stop dispara.

---

## Passo 5 — O efeito das taxas

O teste padrão é **sem taxas** (`fee_pct: 0`) porque a doutrina quer medir o piso bruto. Mas no mercado real, taxas existem. Vamos comparar:

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "fee_pct": 0.001
  }
}
```

### Comparação com e sem taxas (20 momentos)

| Configuração | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|
| Padrão sem fee | −$3.747 | −0.086 | 8/20 |
| Padrão com fee 0.1% | −$24.422 | −0.907 | 1/20 |

O efeito é brutal. Com 0.1% de fee por execução, o EV salta de −0.086 para **−0.907** réguas por momento — mais de 10× pior. Apenas **1 de 20** momentos sobrevive com taxas.

> O teste de sobrevivência com taxas é o **exame de realidade**: se sua estratégia não gera pelo menos +0.9 réguas de edge por trade, ela não supera o custo de execução.

---

## Passo 6 — Mais momentos para convergência

Com 20 momentos, a variância é alta (alguns momentos capturam régua 289, outros 2031). Vamos rodar 50 momentos para ver se o EV converge:

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 50
  }
}
```

### Convergência (sem taxas)

| `momentos` | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|
| 20 | −$3.747 | −0.086 | 8/20 (40%) |
| 50 | −$6.863 | −0.078 | 19/50 (38%) |

O EV converge: de −0.086 para −0.078 réguas por momento. A proporção de pares positivos estabiliza em ~38-40%. Mais momentos dão uma estimativa mais estável do piso, mas a conclusão não muda: **o par não sobrevive sem taxas, e é massacrado com taxas**.

### Convergência (com fee 0.1%)

| `momentos` | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|
| 20 | −$24.422 | −0.907 | 1/20 (5%) |
| 50 | −$40.776 | −0.892 | 4/50 (8%) |

Com taxas, o EV converge para aproximadamente **−0.9 réguas por momento** — estável e implacável.

---

## Anatomia do teste

```
                    ┌──────────────────────────────────────────┐
                    │        ct_testar_sobrevivencia            │
                    ├──────────────────────────────────────────┤
                    │                                          │
                    │  N MOMENTOS espaçados no período         │
                    │  ┌────┐ ┌────┐ ┌────┐     ┌────┐         │
                    │  │ M1 │ │ M2 │ │ M3 │ ...│ MN │         │
                    │  └─┬──┘ └─┬──┘ └─┬──┘     └─┬──┘         │
                    │    │      │      │           │            │
                    │    ▼      ▼      ▼           ▼            │
                    │  CADA MOMENTO dispara:                    │
                    │  ┌──────────────────────────┐             │
                    │  │  COMPRA (gestor adapt.)  │             │
                    │  │  VENDA  (gestor adapt.)  │             │
                    │  │  mesmo lib_grupo, params  │             │
                    │  └────────────┬─────────────┘             │
                    │               ▼                            │
                    │  par_reguas = (pnl_c + pnl_v) / régua     │
                    │               │                            │
                    │               ▼                            │
                    │  AGREGA:                                   │
                    │  • soma_pnl                               │
                    │  • soma_par_reguas                        │
                    │  • ev_par_reguas (soma / n_momentos)      │
                    │  • pares_positivos (par ≥ 0)              │
                    └──────────────────────────────────────────┘
```

### Parâmetros

| Parâmetro | Default | Descrição |
|---|---|---|
| `serie` | — | Série raw (`ct://series/...`) a testar |
| `momentos` | 20 | Número de disparos espaçados no período |
| `stop_r` | 0.5 | Stop em réguas (1 régua = amplitude média de `w_vol`) |
| `lib_grupo` | `"grupo"` | Lib de execução (fork da `grupo`) |
| `fee_pct` | 0 | Taxa por execução (fração do notional). 0 = sem taxas |
| `piramide` | true | Pirâmide 1→2 lotes na confirmação, fora de vol alta |
| `breakeven` | true | Stop → breakeven após andar `stop_r` a favor |
| `reescala_vol` | true | Trailing reescala com a régua corrente |
| `prazo` | 256 | Prazo máximo da posição em barras (fecha a mercado) |
| `w_vol` | 64 | Janela da régua/vol-fato |
| `w_tercil` | 480 | Janela de calibração do tercil de volatilidade |
| `ativacao_r` | 1.0 | Ativação do trailing em réguas a favor |
| `dist_r` | 0.3 | Distância do trailing em réguas |
| `desde_ts` | — | Início do período (unix seconds). Ausente = início da série |
| `ate_ts` | — | Fim do período (unix seconds). Ausente = fim da série |

---

## O que fazer com o resultado

### Cenário A: `ev_par_reguas ≥ 0` (par sobrevive)

O gestor adaptativo sobrevive sem critério de entrada. Isso é **raro e valioso** — significa que a estrutura de stop/take/trailing tem edge estrutural (provavelmente capturando retorno à média ou autocorrelação). Neste caso:

- Adicione um fator de entrada (indicador, filtro, modelo) que melhore o lado — cada acerto vira edge puro.
- O custo das taxas ainda precisa ser superado, mas o piso é positivo.

### Cenário B: `ev_par_reguas < 0` (par sangra) — o caso real

O par sangra sem taxas. É o caso mais comum e não é fracasso — é **informação**. Você sabe exatamente quanto edge precisa gerar:

```
edge_necessário = |ev_par_reguas|  (em réguas por trade)
```

No nosso exemplo: **0.086 réguas/trade** sem taxas, **0.907 réguas/trade** com taxas de 0.1%.

Sua estratégia de entrada precisa adicionar isso para chegar no zero. Qualquer coisa acima disso é lucro.

### Cenário C: `ev_par_reguas << 0` (par massacrado)

Se nem o gestor consegue limitar o dano (EV muito negativo, pouquíssimos pares positivos), o problema é a configuração — não o critério de entrada. Antes de buscar sinais, ajuste:

1. **Encurte o stop** (0.25 régua em vez de 0.5).
2. **Ative breakeven** (a regra que mais pesa).
3. **Desligue pirâmide** (em lado arbitrário, confirmação é ruído).

---

## Próximos passos

- **Adicionar critério de entrada**: combine a lib `grupo` com indicadores para só armar quando houver sinal — veja [RSI + Filtro ADX](./02-rsi-filtro-adx.pt.md).
- **Cross-asset**: veja como diferentes ativos têm pisos de sobrevivência diferentes — veja [Cross-Asset](./05-cross-asset.pt.md).
- **Entender a régua**: a régua é a unidade adaptativa do CT Lab — veja [Lib `grupo`](../docs/05-backtest/05-lib-grupo.pt.md).

---

> Voltar para: [README](../README.md) · [Backtest com `grupo`](./03-lib-grupo-backtest.pt.md) · [Lib `grupo`](../docs/05-backtest/05-lib-grupo.pt.md)

_Last updated: 2026-08-11_
