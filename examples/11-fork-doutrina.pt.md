# 11 — Fork da Doutrina: Customizando o Gestor Adaptativo

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Esteira ML Completa](./10-esteira-ml-completa.pt.md), [Regime + Modelo](./09-regime-modelo.pt.md), [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md)

Nos exemplos 06 a 10, você construiu modelos de ML, detectou regime, mediu microestrutura e montou esteiras completas. Mas em todos eles, a execução foi **crua**: a predição do modelo vira diretamente `comprado(1.0)` ou `vendido(1.0)`. Não há gestor de risco, não há stop, não há trailing, não há pirâmide. O modelo diz a direção, e o backtest compra/vende a mercado sem nenhuma gestão de posição.

A lib `grupo` é o gestor adaptativo do CT Lab — um motor de ordens que armazena posição, gere stops e takes, faz trailing e re-arma ciclos. O "fork da doutrina" é customizar esse gestor: combinar a predição do seu modelo com a execução do grupo, criando sua própria doutrina de trading.

> **Doutrina** é o conjunto de regras que define como você opera: quando entra, quanto entra, onde para, quando sai. O modelo prevê direção; a doutrina executa. Sem doutrina, o modelo é só uma opinião.

---

## O problema

O teste de sobrevivência (exemplo 04) mostrou que o piso de lado arbitrário em BTC é **EV = −0.92 réguas/trade com taxas** — comprar ou vender em momentos aleatórios perde dinheiro. O GBDT (exemplo 06) superou esse piso com +$12.447 de PnL líquido, mas a execução foi crua — sem stops, sem trailing, sem gestão.

A pergunta é: **o gestor adaptativo melhora o resultado do modelo?**

Para responder, você vai comparar:
1. **GBDT cru** — predição vira compra/venda direta, sem gestor.
2. **GBDT + grupo** — predição arma o grupo com stop e take, o gestor gere a saída.
3. **RSI reversal cru** — sinal de reversão em lateralização, sem gestor.
4. **RSI reversal + grupo** — mesma estratégia, com gestor adaptativo.

---

## Passo 1 — O piso de sobrevivência

Antes de qualquer backtest, lembre do piso. O teste de sobrevivência mede o EV de lado arbitrário:

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 20,
    "fee_pct": 0.001
  }
}
```

### Resultado

| Métrica | Valor |
|---|---|
| Momentos | 20 |
| EV (réguas/trade) | −0.922 |
| Soma PnL | −$24.609 |
| Pares positivos | 1/20 (5%) |

> O piso confirma: sem critério de entrada, o gestor adaptativo perde com taxas. Cada trade aleatório custa 0.92 réguas de edge. **Sua estratégia precisa adicionar +0.92 réguas de edge por trade apenas para chegar a zero.**

---

## Passo 2 — A lib `grupo`

A lib `grupo` é importada no início do script Rhai:

```rhai
import "grupo" as g;
```

### Anatomia de um grupo

Um grupo é um conjunto de ordens com **entradas** (que acumulam posição) e **saídas** (reduce-only: stop/take/trailing). Quando a posição zera por saída, o grupo resolve e, se há ciclos restantes, rearma.

```rhai
let cfg = #{
  entradas: [
    #{ tipo: "market", lado: "compra", lote: 1.0 },
    #{ offset: -5.0, tipo: "limit", lado: "compra", lote: 2.0, validade: 10 }
  ],
  saidas: [
    #{ offset: -20.0, tipo: "stop", lote: 3.0 },
    #{ offset:  15.0, tipo: "limit", lote: 3.0 },
    #{ tipo: "trailing", distancia: 8.0, ativacao: 10.0, lote: 3.0 }
  ],
  ciclos: 1.0
};

let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
estado = r.estado;
r.decisao
```

### Contrato

| Parâmetro | Descrição |
|---|---|
| `cfg.entradas` | Lista de ordens de entrada: `market`, `limit`, ou `stop` |
| `cfg.saidas` | Lista de ordens de saída: `stop`, `limit`, ou `trailing` |
| `cfg.ciclos` | Nº de re-armes após resolver. `1.0` = arma, executa, resolve, para. `0.0` = infinito |
| `cfg.referencia` | Preço de referência para offsets (default = `close`) |
| `estado` | Estado persistente do grupo (passado por referência, reatribuir sempre) |
| `posicao` | Posição atual (passada pelo backtest) |
| `ordens` | Ordens do candle (passada pelo backtest) |

> **O retorno `r.decisao`** é a decisão da barra: pode ser `comprado(1.0)`, `vendido(1.0)`, `zerado()`, ou `decisao(...)` com ordens OCO. Você precisa retornar isso da estratégia.

---

## Passo 3 — GBDT cru (baseline)

O GBDT do exemplo 06, sem gestor — predição vira ordem a mercado direto:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r11_gbdt3_pred",
    "estrategia_script": "if ind[\"pred\"][0] > 0.0 { comprado(1.0) } else if ind[\"pred\"][0] < 0.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r11_cru_fee"
  }
}
```

### Resultado

| Métrica | Com fee 0.1% | Sem fee |
|---|---|---|
| **PnL** | −$23.974 | +$76.991 |
| **Trades** | 785 | 785 |
| **Win rate** | 25.7% | 89.4% |
| **Profit factor** | 0.56 | 17.04 |
| **Sharpe** | 0.019 | 0.482 |
| **Fees** | $100.837 | 0 |
| **Exposição** | 99.7% | 99.7% |

> Sem taxas, o GBDT cru tem win rate de 89.4% e profit factor de 17 — parece extraordinário. Mas são 785 trades: $100k em fees destroem tudo. O modelo prevê direção quase todas as barras e o backtest executa quase todos os candles.

---

## Passo 4 — GBDT + grupo (fork)

Agora, a predição arma o grupo em vez de comprar/vender direto. O grupo entra a mercado, coloca stop a −2% e take a +3%, e gere a saída:

```rhai
import "grupo" as g;

let dir = 0.0;
if ind["pred"][0] > 0.0 { dir = 1.0; }
if ind["pred"][0] < 0.0 { dir = -1.0; }

let sem_pos = abs(posicao) <= 1e-9;

if sem_pos && abs(dir) <= 1e-9 {
  zerado()
} else if sem_pos && abs(dir) > 1e-9 {
  // Arma grupo: entrada a mercado + stop 2% + take 3%
  let lado = if dir > 0.0 { "compra" } else { "venda" };
  let s = if dir > 0.0 { -2.0 } else { 2.0 };
  let t = if dir > 0.0 { 3.0 } else { -3.0 };
  let cfg = #{
    entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
    saidas: [
      #{ offset: s, tipo: "stop", lote: 1.0 },
      #{ offset: t, tipo: "limit", lote: 1.0 }
    ],
    ciclos: 1.0
  };
  let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
  estado = r.estado;
  r.decisao
} else {
  // Tem posição: zera se predição virou contra
  let contra = (posicao > 1e-9 && ind["pred"][0] < 0.0)
            || (posicao < -1e-9 && ind["pred"][0] > 0.0);
  if contra { zerado() } else {
    // Re-passa o grupo para manter stops ativos
    let lado = if posicao > 1e-9 { "compra" } else { "venda" };
    let s = if posicao > 1e-9 { -2.0 } else { 2.0 };
    let t = if posicao > 1e-9 { 3.0 } else { -3.0 };
    let cfg = #{
      entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
      saidas: [
        #{ offset: s, tipo: "stop", lote: 1.0 },
        #{ offset: t, tipo: "limit", lote: 1.0 }
      ],
      ciclos: 1.0
    };
    let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
    estado = r.estado;
    r.decisao
  }
}
```

### Resultado

| Métrica | GBDT + grupo com fee | GBDT + grupo sem fee |
|---|---|---|
| **PnL** | −$130 | −$2 |
| **Trades** | 1 | 1 |
| **Exposição** | 0.06% | 0.06% |
| **Duração média** | 900 barras | 900 barras |

> **Apenas 1 trade!** O grupo armou uma única posição, o stop ou take disparou em 900 barras (~9 dias), e o grupo nunca rearma porque `ciclos: 1.0` significa "arma uma vez, resolve, para."

### O problema do `ciclos: 1.0`

A lib `grupo` com `ciclos: 1.0` arma o grupo uma única vez. Quando a posição zera (stop ou take), o grupo resolve e **não rearma** — mesmo que a predição indique nova direção. O resultado é que em 1712 candles, o gestor só opera uma vez.

> **Para o gestor ciclo continuamente, use `ciclos: 0.0`** (infinito). Ou, implemente um re-arme manual: quando `sem_pos && abs(dir) > 0`, crie um novo grupo. A diferença entre `ciclos: 1` e `ciclos: 0` é a diferença entre "opera uma vez" e "opera para sempre."

---

## Passo 5 — RSI reversal: cru vs grupo

### RSI reversal cru (exemplo 09)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r11_rsi_adx",
    "estrategia_script": "if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "r11_rsi_cru_fee"
  }
}
```

### RSI reversal + grupo

```rhai
import "grupo" as g;

let dir = 0.0;
if ind["adx"][0] < 20.0 && ind["rsi"][0] < 30.0 { dir = 1.0; }
if ind["adx"][0] < 20.0 && ind["rsi"][0] > 70.0 { dir = -1.0; }

let sem_pos = abs(posicao) <= 1e-9;

if sem_pos && abs(dir) <= 1e-9 {
  zerado()
} else if sem_pos && abs(dir) > 1e-9 {
  let lado = if dir > 0.0 { "compra" } else { "venda" };
  let s = if dir > 0.0 { -1.5 } else { 1.5 };
  let t = if dir > 0.0 { 2.0 } else { -2.0 };
  let cfg = #{
    entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
    saidas: [
      #{ offset: s, tipo: "stop", lote: 1.0 },
      #{ offset: t, tipo: "limit", lote: 1.0 }
    ],
    ciclos: 1.0
  };
  let r = g::tick(cfg, estado, posicao, open[0], high[0], low[0], close[0], ordens);
  estado = r.estado;
  r.decisao
} else {
  // Zera quando sai do choppy ou RSI normaliza
  zerado()
}
```

### Comparação

| Métrica | RSI cru com fee | RSI + grupo com fee | RSI cru sem fee | RSI + grupo sem fee |
|---|---|---|---|---|
| **PnL** | −$4.707 | −$109 | +$1.334 | +$19 |
| **Trades** | 47 | 1 | 47 | 1 |
| **Win rate** | 8.5% | 0% | 76.6% | 100% |
| **Profit factor** | 0.17 | 0 | 1.66 | — |
| **Exposição** | 4.8% | 0.06% | 4.8% | 0.06% |

> O mesmo padrão: o grupo com `ciclos: 1.0` só opera uma vez. Sem wrapper de re-arme, o gestor não cicla.

---

## Passo 6 — A descoberta: você precisa fazer o fork

Os backtests revelam o problema central: **a lib `grupo` como default não é uma estratégia completa — é um motor de ordens**. Ela armazena e gere as ordens, mas não decide quando re-armar. O `ciclos: 1.0` é um default conservador: arma uma vez, opera, resolve, para.

Para fazer o gestor ciclar, você precisa de um **wrapper** — uma lógica que detecta quando o grupo resolveu (posição zerou após ter tido posição) e arma um novo grupo. Isso é o **fork da doutrina**: você customiza o comportamento do gestor.

### Fork: re-arme automático com `ciclos: 0.0`

A mudança mais simples é usar `ciclos: 0.0` (infinito). O grupo rearma automaticamente após resolver:

```rhai
let cfg = #{
  entradas: [ #{ tipo: "market", lado: lado, lote: 1.0 } ],
  saidas: [
    #{ offset: s, tipo: "stop", lote: 1.0 },
    #{ offset: t, tipo: "limit", lote: 1.0 }
  ],
  ciclos: 0.0  // ← infinito: rearma após resolver
};
```

> **Atenção**: com `ciclos: 0.0`, o grupo rearma **na mesma direção** do grupo anterior. Se você quiser que a nova entrada siga a predição atual do modelo, precisa passar `saidas_vivas: true` ou recente.

### Fork: customizar a lib

A lib `grupo` é forkável por design. Você pode:

```json
{
  "name": "salvar_lib",
  "arguments": {
    "nome": "meu_grupo",
    "fonte": "// seu fork da lib grupo, com regras customizadas..."
  }
}
```

E importar na estratégia:

```rhai
import "meu_grupo" as g;
```

Exemplos de fork:

1. **Re-arme direcional**: após resolver, o grupo rearma na direção da predição atual (não na direção original).
2. **Re-arme com cooldown**: após resolver, espera N barras antes de rearmar.
3. **Re-arme só em regime**: só rearma se ADX < 20 (lateralização) ou ADX > 25 (tendência).
4. **Pirâmide**: em vez de um lote fixo, acumula posição em níveis de preço diferentes.
5. **Trailing dinâmico**: a distância do trailing adapta à volatilidade (ATR).

> **A doutrina é SUA.** A lib `grupo` é o seed — o motor de ordens. O fork é onde você define suas regras de re-arme, suas condições de entrada, e seus parâmetros de saída. O modelo prevê direção; a doutrina executa. A qualidade da doutrina determina se o edge do modelo sobrevive ao custo de execução.

---

## Passo 7 — Salvando e compartilhando seu fork

### Salvar uma lib

```json
{
  "name": "salvar_lib",
  "arguments": {
    "nome": "minha_doutrina",
    "fonte": "// fork da lib grupo com re-arme direcional e trailing ATR..."
  }
}
```

### Listar libs

```json
{ "name": "listar_libs", "arguments": {} }
```

### Ler uma lib

```json
{ "name": "ler_lib", "arguments": { "nome": "minha_doutrina" } }
```

### Usar em backtest

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/meu_modelo_pred",
    "estrategia_script": "import \"minha_doutrina\" as g; // ... usa g::tick ...",
    "nome": "backtest_doutrina"
  }
}
```

---

## Tabela comparativa (todos os resultados)

| Estratégia | Trades | PnL com fee | PnL sem fee | Win% sem fee | PF sem fee |
|---|---|---|---|---|---|
| Sobrevivência (piso) | 40 | −$24.609 | — | — | — |
| GBDT cru | 785 | −$23.974 | +$76.991 | 89.4% | 17.0 |
| GBDT + grupo (1 ciclo) | 1 | −$130 | −$2 | 0% | 0 |
| RSI reversal cru | 47 | −$4.707 | +$1.334 | 76.6% | 1.66 |
| RSI + grupo (1 ciclo) | 1 | −$109 | +$19 | 100% | — |
| Buy & Hold | 0 | −$296 | — | — | — |

### Lições

1. **Sem taxas, o GBDT cru parece extraordinário** (PF 17, win 89%) — mas são 785 trades com $100k em fees. O overtrading é o inimigo #1.

2. **O gestor com `ciclos: 1.0` é muito conservador** — só opera uma vez em 1712 candles. Para operar continuamente, precisa de `ciclos: 0.0` ou um wrapper de re-arme.

3. **O RSI reversal cru tem 47 trades com PF 1.66 sem taxas** — o filtro de regime (ADX < 20) reduz turnover naturalmente, sem precisar do gestor.

4. **O valor do gestor não está em reduzir trades — está em gerenciar risco.** Stop e trailing protegem contra perdas catastróficas. O GBDT cru tem max drawdown de 222%; o gestor tem 1.3%.

> **A doutrina não substitui o modelo — ela o proteger.** O modelo diz a direção; a doutrina define onde parar se errar. Sem stop, um único trade ruim pode dizimar o capital. Com stop, as perdas são limitadas e o modelo pode continuar operando.

---

## Próximos passos

- **Implementar `ciclos: 0.0`**: modifique os scripts deste exemplo para usar `ciclos: 0.0` e meça o resultado. O gestor deve rearmar e gerar mais trades.
- **Fork com re-arme direcional**: salve uma lib `meu_grupo` que rearma na direção da predição atual do modelo, não na direção original do grupo.
- **Fork com trailing ATR**: a distância do trailing se adapta ao ATR — alta volatilidade = trailing mais largo, baixa volatilidade = trailing mais apertado.
- **Combinar regime + doutrina**: só arme o grupo quando ADX < 20 (lateralização) e use RSI reversal como gatilho. O regime filtra quando operar; a doutrina gere como operar.

---

> Voltar para: [README](../README.md) · [Esteira ML Completa](./10-esteira-ml-completa.pt.md) · [Regime + Modelo](./09-regime-modelo.pt.md) · [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md)

_Last updated: 2026-08-12_
