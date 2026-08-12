# 08 — Microestrutura: TFI, OBI, BFI e Backtest

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Modelo GBDT](./06-modelo-gbdt.pt.md), [Modelo MLP](./07-modelo-lstm.pt.md)

Nos exemplos 06 e 07 você treinou modelos de ML para prever a direção do próximo candle — features construídas a partir de indicadores clássicos (RSI, MACD, ADX) sobre OHLCV de 15 minutos. Mas todo OHLCV é uma **agregação**: ele comprime milhares de trades e milhões de updates de book em 4 preços e 1 volume. A informação que se perde na compressão é a **microestrutura** — quem está comprando, quem está vendendo, e com que urgência.

Neste exemplo você vai trabalhar com dados de **microestrutura em 1 segundo** (Premium): o fluxo de taker (trades) e o estado do order book (bid/ask). Vai computar 4 indicadores de microestrutura e analisá-los lado a lado, no mesmo timestamp, para entender como a pressão compradora se manifesta em cada camada do mercado.

> **Microestrutura** é o estudo do processo de formação de preço no nível mais granular — quem inicia a liquidez (maker), quem a toma (taker), e como o desbalanceamento entre os dois revela a direção do fluxo informado.

---

## O problema

Você quer medir **pressão compradora vs vendedora** em tempo real. Existem várias formas de medir:

1. **TFI (Trade Flow Imbalance)** — dos trades executados: `qty_delta / qty`. Se os takers estão comprando mais que vendendo, TFI > 0.
2. **OBI (Order Book Imbalance)** — do top-of-book: `(bid_qty − ask_qty) / (bid_qty + ask_qty)`. Se há mais liquidez de compra no melhor preço, OBI > 0.
3. **BFI (Book Flow Imbalance)** — da mudança no book: `(bid_qty_delta − ask_qty_delta) / (|bid_qty_delta| + |ask_qty_delta|)`. Se o book está sendo preenchido mais no lado comprador, BFI > 0.
4. **DBI (Depth Book Imbalance)** — de camadas profundas: `(depth_bid − depth_ask) / soma`. Igual ao OBI mas olhando ±0.1% ou ±1% do mid.

A diferença entre eles é o **ponto de observação**:

| Indicador | Fonte | Pergunta | Granularidade |
|---|---|---|---|
| TFI | Trades | Quem está **tomando** liquidez? | 1s |
| OBI | Book top | Onde está a **liquidez**? | 1s |
| BFI | Book deltas | Como o book **mudou**? | 1s |
| DBI ±0.1% | Book profundo | Onde está a **parede**? | 1s |

> TFI mede a **ação** (ordens a mercado executadas). OBI/BFI/DBI medem a **intenção** (ordens limite no book). Quando TFI e OBI concordam, a pressão é inequívoca. Quando divergem, algo está acontecendo — talvez absorção.

---

## Passo 1 — Iniciar coletores live (Premium)

O primeiro passo é ligar os coletores de trades e book. Eles operam em 1 segundo e acumulam dados continuamente via WebSocket.

### 1.1 — Coletor de trades

```json
{
  "name": "coletar_trades",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "backfill_dias": 1 }
}
```

```json
{
  "name": "coletar_book",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT" }
}
```

### 1.2 — Verificar status dos coletores

```json
{ "name": "coletas_ativas", "arguments": {} }
```

Ambos retornam um `collector_id` — a URI da série raw:

| Série | URI | Granularidade |
|---|---|---|
| Trades | `ct://series/binance/BTCUSDT/trades_1s` | 1s |
| Book | `ct://series/binance/BTCUSDT/book_1s` | 1s |

> O `backfill_dias: 1` nos trades puxa o histórico de `data.binance.vision` (dump diário) antes de iniciar o WebSocket live. O book não tem backfill — Spot Binance não oferece bulk L2.

---

## Passo 2 — Consultar dados de microestrutura

As séries `trades_1s` e `book_1s` são enormes (centenas de milhares de linhas em 1s). Para inspecionar sem sobrecarga, use `consultar_trades` e `consultar_book` — ambas aceitam `t0`, `t1` e `agregacao`:

### 2.1 — Consultar trades em buckets de 1m

```json
{
  "name": "consultar_trades",
  "arguments": {
    "provider": "binance", "symbol": "BTCUSDT",
    "t0": 1786504560, "t1": 1786505220,
    "agregacao": "1m"
  }
}
```

Resultado (11 buckets de 1 minuto):

| ts | n_trades | qty | qty_delta | TFI | price_open | price_close |
|---|---|---|---|---|---|---|
| 1786504560 | 160 | 0.96 | +0.004 | +0.004 | 63722.00 | 63715.19 |
| 1786504620 | 178 | 5.18 | +3.17 | +0.613 | 63715.20 | 63715.63 |
| 1786504680 | 337 | 7.47 | +0.63 | +0.084 | 63715.63 | 63724.63 |
| 1786504800 | 73 | 1.58 | −0.59 | −0.373 | 63724.63 | 63722.73 |
| 1786504860 | 155 | 3.59 | +2.98 | +0.831 | 63722.72 | 63734.24 |
| 1786504920 | 131 | 4.80 | +4.46 | +0.930 | 63734.23 | 63745.92 |
| 1786504980 | 135 | 5.89 | −0.31 | −0.052 | 63745.92 | 63745.92 |
| 1786505040 | 145 | 1.87 | +1.43 | +0.766 | 63745.92 | 63757.16 |
| 1786505100 | 125 | 1.05 | +0.37 | +0.358 | 63757.16 | 63765.05 |
| 1786505160 | 185 | 4.80 | −3.14 | −0.655 | 63765.04 | 63776.38 |
| 1786505220 | 300 | 3.72 | −3.05 | −0.822 | 63776.37 | 63740.81 |

> **TFI = qty_delta / qty.** No bucket `4920`, qty_delta=+4.46 com qty=4.80 → TFI=+0.93: 93% do volume foi líquido comprador. O preço subiu de 63734 → 63746. TFI funcionou como sinal direcional.

### 2.2 — Consultar book em buckets de 1s (com colunas top-of-book)

```json
{
  "name": "consultar_book",
  "arguments": {
    "provider": "binance", "symbol": "BTCUSDT",
    "t0": 1786505259, "t1": 1786505319,
    "agregacao": "1s",
    "colunas": ["bid_qty_top", "ask_qty_top", "bid_qty_delta", "ask_qty_delta",
                "depth_0_1pct_bid", "depth_0_1pct_ask", "depth_1pct_bid", "depth_1pct_ask",
                "spread_bps", "mid_close", "bid_price", "ask_price"]
  }
}
```

Resultado (primeiros 10 buckets de 1s):

| ts | bid_top | ask_top | OBI | bid_delta | ask_delta | BFI | DBI±0.1% | DBI±1% | mid |
|---|---|---|---|---|---|---|---|---|---|
| 1786505301 | 5.22 | 1.56 | +0.54 | +0.08 | −0.15 | +1.00 | −0.034 | +0.088 | 63740.81 |
| 1786505302 | 5.22 | 0.77 | +0.74 | −0.00 | −1.82 | +0.99 | −0.022 | +0.093 | 63740.81 |
| 1786505303 | 5.24 | 0.25 | +0.91 | −0.00 | −4.03 | +1.00 | +0.096 | +0.126 | 63740.81 |
| 1786505304 | 8.16 | 0.25 | +0.94 | −3.41 | −13.69 | +0.60 | +0.064 | +0.115 | 63747.12 |
| 1786505305 | 8.16 | 0.25 | +0.94 | +2.16 | +2.59 | −0.09 | +0.078 | +0.114 | 63747.12 |
| 1786505306 | 8.22 | 0.25 | +0.94 | −0.08 | +2.32 | −1.00 | +0.070 | +0.111 | 63747.12 |
| 1786505307 | 9.57 | 0.34 | +0.93 | +0.77 | −3.07 | +1.00 | +0.062 | +0.108 | 63751.55 |
| 1786505308 | 9.57 | 0.34 | +0.93 | +2.25 | −0.93 | +1.00 | +0.066 | +0.107 | 63751.55 |
| 1786505309 | 8.39 | 0.20 | +0.95 | +1.87 | −3.27 | +1.00 | +0.093 | +0.117 | 63751.99 |
| 1786505310 | 8.38 | 0.20 | +0.95 | +2.40 | +3.13 | −0.13 | +0.090 | +0.115 | 63751.99 |

> **Leitura:** OBI perto de +0.95 significa que há ~10x mais liquidez de compra que venda no top-of-book. O mid subiu de 63740 → 63752 em 10 segundos. O book está "dizendo" que os makers querem comprar — e o preço respondeu.

---

## Passo 3 — Cruzar TFI × OBI × BFI × DBI no mesmo timestamp

O valor real da microestrutura está em **cruzar** os indicadores. TFI vem dos trades; OBI/BFI/DBI do book. Quando você os alinha no mesmo segundo, vê a dinâmica maker-taker:

```json
{
  "name": "consultar_trades",
  "arguments": {
    "provider": "binance", "symbol": "BTCUSDT",
    "t0": 1786505300, "t1": 1786505360,
    "agregacao": "1s"
  }
}
```

Cruzando manualmente TFI (trades) com OBI/BFI/DBI (book) por timestamp:

| ts | TFI | OBI | BFI | DBI±0.1% | DBI±1% | mid |
|---|---|---|---|---|---|---|
| 1786505301 | +1.00 | +0.539 | +1.00 | −0.034 | +0.088 | 63740.81 |
| 1786505303 | +0.99 | +0.907 | +1.00 | +0.096 | +0.126 | 63740.81 |
| 1786505304 | +1.00 | +0.940 | +0.60 | +0.064 | +0.115 | 63747.12 |
| 1786505307 | +0.98 | +0.931 | +1.00 | +0.062 | +0.108 | 63751.55 |
| 1786505309 | +0.79 | +0.953 | +1.00 | +0.093 | +0.117 | 63751.99 |
| 1786505315 | −0.39 | +0.886 | −1.00 | +0.050 | +0.097 | 63755.58 |
| 1786505316 | −0.82 | +0.844 | −1.00 | +0.040 | +0.094 | 63755.58 |
| 1786505319 | −1.00 | +0.953 | +0.94 | +0.045 | +0.095 | 63755.58 |
| 1786505352 | −1.00 | −0.324 | −0.93 | −0.087 | +0.084 | 63756.58 |
| 1786505353 | −1.00 | +0.269 | +0.04 | −0.040 | +0.082 | 63756.58 |

> **Divergência em 1786505315–316:** TFI virou negativo (taker vendendo) mas OBI continua +0.88 (book com mais liquidez de compra). Isso é **absorção** — alguém está vendendo a mercado contra uma parede de compras. Quando a parede cede (ts 5352: OBI vira −0.32), o preço cai.

---

## Passo 4 — Computar indicadores como séries derivadas

As tools `tfi`, `obi`, `bfi`, `dbi01`, `dbi1` computam cada indicador sobre a série raw e persistem como série derivada. O `ct_tfi`, `ct_obi`, `ct_bfi` fazem a versão VWMA windowed (ponderada por qty/atividade):

### 4.1 — TFI instantâneo e windowed

```json
{
  "name": "tfi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/trades_1s" }
}
```

```json
{
  "name": "ct_tfi",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/trades_1s",
    "period": 15,
    "name": "btc_tfi_15s"
  }
}
```

> `tfi` computa `qty_delta / qty` por bucket de 1s. `ct_tfi` faz a VWMA windowed: `Σ(tfi_i · qty_i) / Σ(qty_i)` na janela de 15 segundos — buckets com mais volume pesam mais.

### 4.2 — OBI, BFI, DBI do book

```json
{ "name": "obi",  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

```json
{ "name": "bfi",  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

```json
{ "name": "dbi01", "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

```json
{ "name": "dbi1",  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s" } }
```

### 4.3 — Versões windowed (VWMA)

```json
{
  "name": "ct_obi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s", "period": 15, "name": "btc_obi_15s" }
}
```

```json
{
  "name": "ct_bfi",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/book_1s", "period": 15, "name": "btc_bfi_15s" }
}
```

> As versões windowed suavizam o ruído de 1s. No OBI, 15s de VWMA ponderada por `(bid_qty_top + ask_qty_top)` dá mais peso a buckets com mais liquidez — o "consenso" do book.

---

## Passo 5 — Backtest com BOP (proxy OHLCV do fluxo)

As séries de microestrutura (trades_1s, book_1s) estão em 1 segundo. O `ct_backtest` opera sobre OHLCV em timeframes maiores (ex: 15m). Não é possível fazer join direto entre 1s e 15m via `compor_serie` — o timeframe é incompatível.

A solução é usar **BOP (Balance of Power)** — o análogo OHLCV do TFI. BOP mede a pressão compradora dentro de cada candle: `(close − open) / (high − low)`. Está em [-1, +1] e é computável diretamente sobre a série de preço, sem precisar de trades.

### 5.1 — Materializar ct_bop (windowed)

```json
{
  "name": "ct_bop",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "period": 14,
    "name": "btc_ct_bop_14"
  }
}
```

> `ct_bop` computa BOP por candle e faz VWMA windowed de 14 períodos, ponderada por volume. Resultado: `ct://derived/btc_ct_bop_14` com 1712 linhas (mesmo timeframe da série 15m).

### 5.2 — Backtest com threshold 0.2

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/btc_ct_bop_14",
    "estrategia_script": "if ind[\"ct_bop\"][0] > 0.2 { comprado(1.0) } else if ind[\"ct_bop\"][0] < -0.2 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "tfi_bop_fee"
  }
}
```

### 5.3 — Resultados

| Métrica | BOP com fee 0.1% | BOP sem fee | Buy & Hold |
|---|---|---|---|
| **PnL** | −$16.178 | +$398 | −$296 |
| **PnL bruto** | +$398 | +$398 | 0 |
| **Fees** | $16.576 | 0 | 0 |
| **Trades** | 129 | 129 | 0 |
| **Win rate** | 11.6% | 35.7% | — |
| **Profit factor** | 0.20 | 1.06 | 0 |
| **Expectativa** | −$125/trade | +$3.1/trade | 0 |
| **Payoff** | 1.50 | 1.91 | — |
| **Exposição** | 27.5% | 27.5% | 99.9% |
| **Sharpe** | −0.014 | 0.007 | 0.003 |
| **Sortino** | −0.016 | 0.012 | 0.004 |
| **Calmar** | −0.62 | 6.94 | −1.59 |
| **Max DD** | 161.3% | 17.7% | 28.9% |
| **Retorno anual.** | −100% | +122.6% | −45.9% |

### Análise

**Sem taxas, o BOP tem edge positivo.** PnL de +$398, profit factor 1.06, win rate 35.7%, e Calmar de 6.94 — o drawdown máximo é apenas 17.7%. A expectativa por trade é +$3.1 — cada trade, em média, adiciona 0.03% ao capital.

**Com taxas de 0.1%, o edge é destruído.** As fees comem $16.576 — 42x o PnL bruto. O win rate despenca de 35.7% para 11.6% não porque o número de acertos mudou (mesmos 129 trades), mas porque as taxas transformam trades que eram marginalmente vencedores em perdedores. A expectativa vira −$125/trade.

> **O problema é o turnover.** 129 trades em 1712 candles (7.5% dos candles) com exposição de 27.5% é muito para BOP em 15m. Cada entrada+saída custa 0.2% do notional — 129 trades × 2 lados × 0.1% = $16.576 em fees sobre $10.000 de capital.

### 5.4 — Threshold maior reduz trades mas não resolve

| Threshold | Trades | PnL com fee | PnL bruto | Fees | Win rate |
|---|---|---|---|---|---|
| 0.2 | 129 | −$16.178 | +$398 | $16.576 | 11.6% |
| 0.3 | 50 | −$5.842 | +$579 | $6.421 | 10.0% |

Threshold 0.3 reduz trades de 129 para 50 e melhora o PnL bruto (+$579 vs +$398), mas as fees ainda consomem tudo. Falta edge suficiente para superar o custo de execução.

---

## Passo 6 — Por que BOP não supera taxas (e TFI seria pior)

BOP é um **proxy** do TFI construído a partir de OHLCV. O TFI real tem dois agravantes:

1. **Sinal em 1s é muito mais frequente** — cada segundo de trades gera um sinal. Seriam milhares de trades por dia no backtest, com fees proporcionais.
2. **TFI é extremo por design** — em buckets de 1s com poucos trades, TFI é frequentemente ±1.0 (todos os trades são de um lado). Isso gera sinais quase contínuos, com altíssimo turnover.

A questão estrutural é: **microestrutura tem muito mais sinais que OHLCV, mas cada sinal carrega menos edge preditivo**. O conjunto TFI+OBI converge para a mesma direção que BOP — mas com 100x mais trades.

> É por isso que microestrutura é usada para **execução** (TWAP/VWAP, timing de entrada/saída, detecção de absorção) e não para **sinal discrecional** em 15m. O TFI brilhantemente detecta quando há pressão compradora neste segundo — mas não prevê se o preço vai subir nos próximos 15 minutos.

---

## Quando microestrutura ajuda

| Cenário | Como usar |
|---|---|
| **Timing de entrada** | TFI > 0.8 confirma pressão compradora — entra na compra com confiança |
| **Detecção de absorção** | TFI negativo + OBI positivo = vendedores sendo absorvidos — possível reversão |
| **Confirmação de breakout** | Preço rompe nível + BFI > 0.5 = book está apostando na direção |
| **Filtro de falsos sinais** | BOP diz comprar mas TFI < 0 = divergência — não entra |
| **Slippage estimation** | Spread + depthDoBook → estima custo de execução antes de entrar |

> **Microestrutura não substitui a estratégia — ela a informa.** Use TFI/OBI/BFI como **filtro** sobre sinais de um modelo (como o GBDT do exemplo 06), não como sinal standalone.

---

## Próximos passos

- **Combinar BOP + GBDT**: use o BOP como feature adicional no pipeline do exemplo 06 — o modelo aprende a interpretar a pressão compradora junto com RSI/MACD/ADX.
- **Regime detection**: use TFI + BOP para detectar se o mercado está em regime direcional ou lateral — veja [Regime + Modelo](./09-regime-modelo.pt.md).
- **Fork da doutrina**: use microestrutura como filtro do gestor adaptativo — entra apenas quando TFI e OBI convergem — veja [Fork da Doutrina](./11-fork-doutrina.pt.md).
- **Live timing**: use `consultar_trades` e `consultar_book` em tempo real para timing de execução dentro do gestor adaptativo.

---

> Voltar para: [README](../README.md) · [Modelo GBDT](./06-modelo-gbdt.pt.md) · [Modelo MLP](./07-modelo-lstm.pt.md) · [Regime + Modelo](./09-regime-modelo.pt.md)

_Last updated: 2026-08-12_
