# 05 — Cross-Asset: O Piso Varia com o Ativo

> **Nível:** Intermediário · **Premium** · **Pré-requisitos:** [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md), [Lib `grupo`](../docs/05-backtest/05-lib-grupo.pt.md)

No [exemplo 04](./04-teste-sobrevivencia.pt.md) você rodou o teste de sobrevivência em BTCUSDT e descobriu que o par sangra — o gestor adaptativo não sobrevive sem critério de entrada. Mas será que isso é universal? Será que **todos os ativos sangram igual**?

A resposta é não. O piso de sobrevivência **varia dramaticamente** entre ativos. Alguns ativos têm estrutura de retorno à média tão forte que o par sobrevive até sem taxas — e às vezes até **com taxas**. Outros sangram tanto que nem o melhor gestor do mundo salva.

Neste exemplo, você vai:

1. **Buscar dados de 5 ativos** — BTC, ETH, SOL, BNB e XRP em 15m.
2. **Rodar o teste de sobrevivência** em cada um, com os mesmos parâmetros.
3. **Comparar os pisos** — ver quais ativos sobrevivem e quais sangram.
4. **Adicionar taxas** e ver quais ativos ainda sobrevivem.
5. **Entender o porquê** — o que torna um ativo sobrevivente.
6. **Convergência** com 50 momentos para validar a leitura.

---

## O problema

Você está decidindo em quais ativos vai operar. Sua estratégia usa a lib `grupo` com stop de 0.5 régua, breakeven e trailing — o gestor adaptativo padrão da doutrina. Antes de gastar meses procurando um sinal de entrada, você quer saber: **qual ativo dá à sua estratégia o maior piso de sobrevivência?**

O teste de sobrevivência cross-asset responde isso em minutos. Você roda o mesmo teste em múltiplos ativos e compara os `ev_par_reguas` — o valor esperado por momento em réguas. Ativos com EV positivo têm sobrevida. Ativos com EV muito negativo precisam de mais edge para compensar.

> A régua é a unidade adaptativa do CT Lab: 1 régua = amplitude média da janela `w_vol` (64 barras por padrão). O stop de 0.5 régua se ajusta à volatilidade corrente de **cada ativo** — em BTC, 1 régua pode ser $1.200; em XRP, pode ser $0.01. A normalização permite comparar ativos de preços completamente diferentes.

---

## Passo 1 — Buscar dados de 5 ativos

```
Busque BTCUSDT, ETHUSDT, SOLUSDT, BNBUSDT e XRPUSDT em 15m na Binance.
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "ETHUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "SOLUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "BNBUSDT", "interval": "15m" }
}
```

```json
{
  "name": "buscar_serie",
  "arguments": { "provider": "binance", "symbol": "XRPUSDT", "interval": "15m" }
}
```

> Cada série fica disponível em `ct://series/binance/<SYMBOL>/15m`.

---

## Passo 2 — Rodar o teste em cada ativo (20 momentos, sem taxas)

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": { "serie": "ct://series/binance/BTCUSDT/15m" }
}
```

Repita para cada ativo, trocando o `symbol` na URI.

### Resultado comparativo (20 momentos, sem taxas)

| Ativo | Preço aprox. | Régua aprox. | `soma_pnl` | `ev_par_reguas` | `pares_positivos` |
|---|---|---|---|---|---|
| **BTC** | ~$110k | ~$1.200 | −$3.807 | **−0.095** | 7/20 (35%) |
| **ETH** | ~$3.500 | ~$45 | −$263 | **−0.449** | 4/20 (20%) |
| **SOL** | ~$180 | ~$2 | +$9 | **+0.254** | 16/20 (80%) |
| **BNB** | ~$550 | ~$6 | +$246 | **+2.054** | 20/20 (100%) |
| **XRP** | ~$0.50 | ~$0.01 | −$0.19 | **−0.337** | 4/20 (20%) |

### A leitura

Os resultados revelam **três classes de ativos**:

1. **BNB — sobrevivente absoluto**: EV de +2.05 réguas por momento, 20/20 pares positivos. O gestor adaptativo não só sobrevive — ele **ganha** sem critério de entrada. Isso sugere retorno à média muito forte no período testado.

2. **SOL — sobrevivente marginal**: EV de +0.254, 16/20 positivos. O par sobrevive, mas com folga pequena. Qualquer custo real (slippage, spread) pode comer essa margem.

3. **BTC, ETH, XRP — não-sobreviventes**: Todos têm EV negativo. BTC é o "menos pior" (−0.095), XRP e ETH são os piores (−0.337 e −0.449). Estes ativos precisam de um fator de entrada que adicione edge para compensar.

> **Atenção**: estes resultados são específicos para o período amostrado (1000 candles de 15m) e a configuração padrão do gestor. Não são universais — o piso de um ativo muda com o regime de mercado. O valor do teste cross-asset está na **comparação relativa**: qual ativo dá mais folga para sua estratégia.

---

## Passo 3 — O efeito das taxas por ativo

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": { "serie": "ct://series/binance/BNBUSDT/15m", "fee_pct": 0.001 }
}
```

### Resultado comparativo (20 momentos, com fee 0.1%)

| Ativo | `soma_pnl` (sem fee) | `soma_pnl` (com fee) | `ev_par_reguas` (com fee) | `pares_positivos` (com fee) |
|---|---|---|---|---|
| **BTC** | −$3.807 | −$24.609 | **−0.922** | 1/20 (5%) |
| **ETH** | −$263 | −$451 | **−0.832** | 0/20 (0%) |
| **SOL** | +$9 | −$15 | **−0.361** | 10/20 (50%) |
| **BNB** | +$246 | +$186 | **+1.580** | 19/20 (95%) |
| **XRP** | −$0.19 | −$0.28 | **−0.495** | 1/20 (5%) |

### A leitura com taxas

O cenário muda drasticamente:

- **BNB** é o **único** que sobrevive com taxas: EV de +1.58 réguas, 19/20 pares positivos. O edge estrutural é tão forte que absorve 0.1% de fee por execução e ainda sobra.
- **SOL** perdeu o piso: caiu de +0.254 para −0.361. A folga era pequena e as taxas comeram tudo.
- **BTC, ETH, XRP** pioram proporcionalmente. ETH é o caso mais extremo: **0/20** pares positivos com taxas — nenhum momento sobrevive.

> O teste com taxas é o **filtro final**. Ativos que sobrevivem sem taxas mas morrem com taxas (como SOL) precisam de um fator de entrada que adicione pelo menos +0.361 réguas de edge. BNB já tem edge positivo líquido — qualquer fator de entrada é lucro puro.

---

## Passo 4 — Por dentro: BNB vs BTC

Vamos olhar `por_momento` para entender **por que** BNB sobrevive e BTC não.

### BNB — o sobrevivente (20 momentos, sem taxas)

| Momento | Régua | PnL compra | PnL venda | `par_reguas` |
|---|---|---|---|---|
| 1 | 9 | +8 | −5 | +0.31 |
| 2 | 9 | +22 | −5 | +1.81 |
| 3 | 9 | +21 | −5 | +1.77 |
| 4 | 9 | +14 | −5 | +0.99 |
| 5 | 9 | +13 | −5 | +0.88 |
| ... | ... | ... | ... | ... |
| 12 | 7 | +24 | −3 | +3.23 |
| 13 | 5 | +25 | −2 | +5.12 |
| 14 | 4 | +20 | −2 | +4.51 |
| ... | ... | ... | ... | ... |
| 20 | 6 | +10 | −3 | +1.14 |

**Padrão**: Em BNB, o lado compra **consistentemente captura mais do que o lado venda perde**. A régua é pequena (4-9), o stop é apertado, e o ativo tem forte retorno à média — o preço sobe e cai dentro do canal, e o gestor captura o movimento de ambos os lados. O `par_reguas` é positivo em **todos os 20 momentos**.

### BTC — o não-sobrevivente (20 momentos, sem taxas)

| Momento | Régua | PnL compra | PnL venda | `par_reguas` |
|---|---|---|---|---|
| 1 | 1233 | 0 | −617 | −0.50 |
| 2 | 1548 | −774 | +280 | −0.32 |
| 3 | 2031 | −329 | +329 | 0.00 |
| 4 | 1383 | −680 | 0 | −0.49 |
| 5 | 289 | −145 | +803 | +2.28 |
| ... | ... | ... | ... | ... |
| 12 | 922 | 0 | −461 | −0.50 |
| 13 | 934 | −467 | 0 | −0.50 |
| ... | ... | ... | ... | ... |
| 20 | 866 | 0 | −433 | −0.50 |

**Padrão**: Em BTC, a régua é enorme (289-2031) e o `par_reguas` frequentemente é **−0.50** — o valor do stop puro. Isso significa: um lado foi estopado e o outro não executou nada. O movimento direcional do BTC no período foi grande o suficiente para que, em muitos momentos, o gestor não conseguisse capturar ambos os lados — um lado toma stop e o outro fica parado.

### A diferença estrutural

| Característica | BNB | BTC |
|---|---|---|
| Régua típica | 4–9 | 600–2000 |
| `par_reguas` dominante | +0.3 a +5.0 (positivo) | −0.50 (stop puro) |
| Movimento | Canal (retorno à média) | Tendência direcional |
| Gestor captura | Ambos os lados | Um lado toma stop |

BNB tem **régua curta e canal** — o gestor adaptativo prospera porque o preço reverte dentro do range. BTC tem **régua longa e tendência** — o gestor sofre porque o preço vai embora e estopa um lado.

---

## Passo 5 — Convergência com 50 momentos

Com 20 momentos, a variância é alta. Vamos rodar 50 momentos para confirmar que os resultados não são ruído:

### Convergência (50 momentos, sem taxas)

| Ativo | `soma_pnl` (20) | `soma_pnl` (50) | `ev_par_reguas` (20) | `ev_par_reguas` (50) | `pares_positivos` (50) |
|---|---|---|---|---|---|
| BTC | −$3.807 | −$8.769 | −0.095 | −0.128 | 18/50 (36%) |
| ETH | −$263 | −$499 | −0.449 | −0.352 | 15/50 (30%) |
| SOL | +$9 | +$26 | +0.254 | +0.296 | 42/50 (84%) |
| BNB | +$246 | +$614 | +2.054 | +2.078 | 50/50 (100%) |
| XRP | −$0.19 | −$0.51 | −0.337 | −0.363 | 7/50 (14%) |

### Convergência (50 momentos, com fee 0.1%)

| Ativo | `soma_pnl` (50) | `ev_par_reguas` (50) | `pares_positivos` (50) |
|---|---|---|---|
| BTC | −$46.943 | −1.093 | 4/50 (8%) |
| ETH | −$922 | −0.680 | 0/50 (0%) |
| SOL | −$45 | −0.408 | 24/50 (48%) |
| BNB | +$467 | +1.602 | 49/50 (98%) |
| XRP | −$0.73 | −0.521 | 0/50 (0%) |

### O que a convergência confirma

1. **BNB é estruturalmente sobrevivente**: 50/50 pares positivos sem taxas, 49/50 com taxas. O EV converge para +2.08 sem fee e +1.60 com fee — estável e robusto.

2. **SOL confirma o piso marginal**: EV +0.30 sem taxas (ligeiramente melhor que 20 momentos), mas −0.41 com taxas. A folga sem taxas é real, mas pequena demais para absorver custo de execução.

3. **BTC piora levemente com mais momentos**: EV de −0.095 para −0.128. A tendência direcional é mais consistente do que sugeriam 20 momentos.

4. **ETH e XRP são os piores**: ETH tem 0/50 com taxas, XRP também. Ambos precisam de edge substancial antes de serem operáveis.

---

## Passo 6 — O que torna um ativo sobrevivente?

O teste cross-asset revela que a sobrevivência depende da **estrutura de microestrutura do ativo**:

| Característica | Sobrevivente (BNB, SOL) | Não-sobrevivente (BTC, ETH, XRP) |
|---|---|---|
| **Régua** | Curta (4-9 em BNB) | Longa (600-2000 em BTC) |
| **Movimento** | Canal / retorno à média | Tendência direcional |
| **Autocorrelação** | Negativa (reversão) | Positiva (momentum) |
| **Stop adaptativo** | Flagging rápido, captura reversão | Stop largo, perde na direção |
| **Lado arbitrário** | Ganha — ambos os lados capturam | Perde — um lado toma stop |

### Por que BNB tem régua curta?

A régua é a amplitude média da janela `w_vol` (64 barras). BNB em 15m tem baixa volatilidade relativa no período — movimentos de $4-$9 por barra. Com stop de 0.5 régua, o gestor opera em um canal de $2-$5. Quando o preço reverte (e BNB tem forte reversão em 15m), o lado oposto captura rapidamente.

### Por que BTC tem régua longa?

BTC move $600-$2000 por barra em 15m. Com stop de 0.5 régua, o gestor precisa de movimentos de $300-$1000 para capturar. Em períodos de tendência forte, o preço vai embora antes de reverter — um lado é estopado e o outro não tem tempo de executar.

> **Não é que BTC seja "pior" que BNB** — é que a **estrutura do gestor adaptativo** (stop + breakeven + trailing) favorece ativos com retorno à média. Em um ativo de tendência, o gestor precisa de um **fator de entrada** que filtre o momentum para evitar operar contra a tendência.

---

## Como usar o resultado cross-asset

### Seletor de ativos

Antes de desenvolver uma estratégia, rode o teste de sobrevivência em uma cesta de ativos. Escolha os que têm **EV mais próximo de zero ou positivo** — estes dão a maior folga para sua estratégia.

```
ranking = ordenar_ativos_por(ev_par_reguas)
# BNB (+2.05) > SOL (+0.25) > BTC (−0.095) > XRP (−0.337) > ETH (−0.449)
```

### Dimensionamento de edge

Para cada ativo, o `|ev_par_reguas|` diz **quanto edge você precisa gerar**:

| Ativo | Edge necessário (sem fee) | Edge necessário (com fee 0.1%) |
|---|---|---|
| BNB | 0 (já é positivo) | 0 (já é positivo) |
| SOL | 0 (já é positivo) | 0.361 réguas/trade |
| BTC | 0.095 réguas/trade | 0.922 réguas/trade |
| XRP | 0.337 réguas/trade | 0.495 réguas/trade |
| ETH | 0.449 réguas/trade | 0.832 réguas/trade |

- **BNB**: qualquer fator de entrada vira lucro líquido — mesmo com taxas.
- **ETH**: precisa gerar 0.83 réguas de edge por trade só para chegar no zero. É muito.
- **BTC**: precisa de 0.92 réguas com taxas — difícil, mas factível com um bom modelo.

### Caveatas

1. **Período importa**: os resultados são específicos para os 1000 candles amostrados. Em um período diferente (bull market vs bear market), o ranking pode mudar.
2. **Timeframe importa**: BTC em 1h pode ter régua mais curta (menor ruído) e melhor sobrevivência.
3. **Regime importa**: BNB sobreviver em canal não significa que sobrevive em trend. Rode o teste periodicamente.
4. **O teste não substitui backtest**: o teste de sobrevivência mede o **piso** — o limite inferior. Sua estratégia real (com critério de entrada) deve fazer melhor que o piso.

---

## Próximos passos

- **Estratégia com critério de entrada**: combine o gestor com indicadores para operar apenas sobreviventes — veja [RSI + Filtro ADX](./02-rsi-filtro-adx.pt.md).
- **Testar outros timeframes**: rode o teste em 1h ou 4h e veja se o ranking muda — veja [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md).
- **Modelo de ML**: treine um modelo para prever lado em ativos com piso negativo — veja [Modelo GBDT](./06-modelo-gbdt.pt.md).

---

> Voltar para: [README](../README.md) · [Teste de Sobrevivência](./04-teste-sobrevivencia.pt.md) · [Lib `grupo`](../docs/05-backtest/05-lib-grupo.pt.md)

_Last updated: 2026-08-11_
