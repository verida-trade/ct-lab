# 09 — Regime: Quando o Mercado Muda de Personalidade

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Microestrutura TFI](./08-tfi-backtest.pt.md), [Modelo GBDT](./06-modelo-gbdt.pt.md)

O GBDT do exemplo 06 gerou +$12.447 de lucro líquido com taxas. O BOP do exemplo 08 teve edge positivo sem taxas (+$398) mas foi destruído por taxas (−$16.178). A pergunta que não quer calar: **esses resultados valem para todo o período, ou só para metade dele?**

Todo mercado alterna entre **regimes** — fases com características estatísticas distintas. Num regime de tendência, o preço persiste numa direção: o que subiu ontem tende a subir hoje. Num regime lateral, o preço reverte: o que subiu ontem tende a cair hoje. Se você não sabe em qual regime está, sua estratégia pode estar perfeitamente adaptada ao regime errado.

Neste exemplo você vai:
1. **Medir a estrutura** do caminho de preço — variance ratio e curtose por escala.
2. **Detectar regime** usando ADX (tendência) vs RSI (lateralização).
3. **Comparar estratégias** em cada regime — trend-following vs mean reversion.
4. **Combinar regime + sinal** num backtest com filtro de regime dinâmico.

---

## O problema

Uma estratégia é uma aposta sobre uma propriedade estatística do preço:

- **Trend-following** aposta que `Var(pT) / T > Var(p1)` — o preço persiste. Funciona quando VR > 1.
- **Mean reversion** aposta que `Var(pT) / T < Var(p1)` — o preço reverte. Funciona quando VR < 1.

Se você roda trend-following num mercado lateral (VR < 1), você compra no topo e vende no fundo — exatamente o oposto do que deveria fazer. O problema é que **o regime muda com o tempo** e você precisa detectar quando.

> **Variance Ratio (VR)** mede se o preço é um random walk. VR = 1 significa random walk (sem memória). VR > 1 significa persistência (tendência). VR < 1 significa anti-persistência (reversão). É a métrica mais fundamental de regime.

---

## Passo 1 — Medir a estrutura do caminho de preço

A tool `ct_medir_estrutura` computa variance ratio e curtose por escala (1, 4, 8, ..., 512 barras), subdivididos por tercil de volatilidade e por bloco temporal:

```json
{
  "name": "ct_medir_estrutura",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m"
  }
}
```

### 1.1 — Variance Ratio geral por escala

| Escala (τ) | VR | Curtose | Interpretação |
|---|---|---|---|
| 1 | 1.00 | 6.86 | (referência) |
| 4 | 0.94 | 8.24 | Anti-persistência leve |
| 8 | 0.96 | 5.18 | ≈ random walk |
| 16 | 0.99 | 2.27 | ≈ random walk |
| 32 | 0.97 | 1.34 | ≈ random walk |
| 64 | 0.74 | 1.63 | Anti-persistência forte |
| 128 | 0.58 | 0.19 | Anti-persistência forte |
| 256 | 0.42 | −0.32 | Anti-persistência muito forte |
| 512 | 0.29 | −0.72 | Anti-persistência extrema |

> **Em escala curta (τ=1–32), o preço é praticamente random walk** (VR ≈ 1). Em escala longa (τ=64–512), **forte anti-persistência** (VR < 1). Isso significa: no curto prazo o preço não tem direção preferida, mas em janelas longas ele **reverte** — o que sobe muito tende a cair. É um mercado que **não tem tendência sustentada**.

### 1.2 — VR por tercil de volatilidade

| Escala | Vol baixa | Vol média | Vol alta |
|---|---|---|---|
| 4 | 0.80 | 0.73 | 1.22 |
| 8 | 0.77 | 0.68 | 1.34 |
| 16 | 0.85 | 0.67 | 1.23 |
| 32 | 0.98 | 1.02 | 0.81 |
| 64 | 0.58 | 1.14 | 0.46 |

> **Em volatilidade alta, VR > 1 em escalas curtas** (τ=4–16): o preço persiste — há movimento direcional com volatilidade. Em volatilidade baixa, VR < 1: o preço oscila lateralmente. A conclusão é que **regime de volatilidade determina persistência**.

### 1.3 — VR por bloco temporal (8 blocos de ~36h cada)

| Período | VR(τ=4) | VR(τ=16) | Regime |
|---|---|---|---|
| Bloco 1 (10–11 ago) | 1.26 | 0.93 | Tendência de curto prazo |
| Bloco 2 (11–13 ago) | 0.80 | 1.19 | Lateral → tendência |
| Bloco 3 (13–14 ago) | 1.07 | 1.32 | Tendência |
| Bloco 4 (14–15 ago) | 0.65 | 0.29 | Forte lateralização |
| Bloco 5 (15–17 ago) | 0.97 | 1.01 | Random walk |
| Bloco 6 (17–18 ago) | 0.53 | 0.35 | Forte lateralização |
| Bloco 7 (18–20 ago) | 0.71 | 0.98 | Lateral |
| Bloco 8 (20–21 ago) | 1.05 | 1.34 | Tendência |

> **O regime muda de bloco para bloco.** Bloco 3 era tendência (VR > 1), bloco 4 era lateral (VR = 0.29), bloco 8 voltou a tendência. Uma estratégia fixa seria destruída metade do tempo.

---

## Passo 2 — Detectar regime com ADX

Variance Ratio é uma análise post-hoc — você não pode usá-lo em tempo real porque precisa de centenas de barras. O **ADX** (Average Directional Index) é um proxy em tempo real: mede a força da tendência nos últimos 14 candles.

### 2.1 — Materializar ADX

```json
{
  "name": "adx",
  "arguments": { "uri": "ct://series/binance/BTCUSDT/15m", "period": 14, "name": "btc_adx_regime" }
}
```

### 2.2 — Construir regime features + BOP + RSI num pipeline único

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "r09_bop_adx_rsi",
    "output": "$features",
    "steps": [
      { "op": "bop", "id": "bop", "source": "$anchor" },
      { "op": "adx", "id": "adx", "source": "$anchor", "period": 14 },
      { "op": "rsi", "id": "rsi", "source": "$anchor", "column": "close", "period": 14 },
      {
        "op": "compose",
        "id": "features",
        "columns": [
          { "as_column": "bop", "source": "$bop", "source_column": "bop" },
          { "as_column": "adx", "source": "$adx", "source_column": "adx" },
          { "as_column": "plus_di", "source": "$adx", "source_column": "plus_di" },
          { "as_column": "minus_di", "source": "$adx", "source_column": "minus_di" },
          { "as_column": "rsi", "source": "$rsi", "source_column": "rsi" }
        ]
      }
    ]
  }
}
```

> A série `ct://derived/r09_bop_adx_rsi` contém BOP, ADX, +DI, −DI, e RSI em 15m — tudo alinhado no mesmo timestamp.

---

## Passo 3 — Backtests por regime

Agora você roda 4 estratégias e compara. Todas usam a mesma série composta `r09_bop_adx_rsi` como indicadores:

### 3.1 — BOP base (sem filtro de regime)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"bop\"][0] > 0.2 { comprado(1.0) } else if ind[\"bop\"][0] < -0.2 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_bop_base_fee"
  }
}
```

### 3.2 — BOP filtrado por ADX > 25 (apenas tendência)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"bop\"][0] > 0.2 && ind[\"adx\"][0] > 25.0 { comprado(1.0) } else if ind[\"bop\"][0] < -0.2 && ind[\"adx\"][0] > 25.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_bop_trend_fee"
  }
}
```

### 3.3 — BOP filtrado por ADX < 20 (apenas lateralização)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"bop\"][0] > 0.2 && ind[\"adx\"][0] < 20.0 { comprado(1.0) } else if ind[\"bop\"][0] < -0.2 && ind[\"adx\"][0] < 20.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_bop_choppy_fee"
  }
}
```

### 3.4 — RSI reversal em lateralização (ADX < 20, RSI < 30 ou > 70)

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "indicadores": "ct://derived/r09_bop_adx_rsi",
    "estrategia_script": "if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"adx\"][0] < 20.0 && ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "capital_inicial": 10000,
    "fee_pct": 0.001,
    "nome": "regime_rsi_chop_fee"
  }
}
```

---

## Passo 4 — Resultados

### Sem taxas (edge bruto)

| Estratégia | Regime | Trades | PnL | Win% | PF | Sharpe | Calmar | Max DD |
|---|---|---|---|---|---|---|---|---|
| BOP base | Todos | 866 | −$3.649 | 38.3% | 0.88 | −0.022 | −2.47 | 40.4% |
| BOP ADX>25 | Tendência | 418 | −$2.238 | 36.1% | 0.84 | −0.021 | −3.68 | 27.1% |
| BOP ADX<20 | Lateral | 284 | +$804 | 43.0% | 1.08 | 0.011 | 19.6 | 19.8% |
| RSI reversal (ADX<20) | Lateral | 47 | +$1.334 | 76.6% | 1.66 | 0.027 | 133.3 | 9.0% |
| RSI trend (ADX>25) | Tendência | 174 | +$246 | 40.8% | 1.03 | 0.005 | 5.01 | 12.9% |
| Buy & Hold | — | 0 | −$296 | — | — | 0.003 | −1.59 | 28.9% |

### Com taxas 0.1%

| Estratégia | Regime | Trades | PnL | Win% | PF | Sharpe | Calmar |
|---|---|---|---|---|---|---|---|
| BOP base | Todos | 866 | −$114.880 | 7.2% | 0.074 | 0.025 | −0.087 |
| BOP ADX>25 | Tendência | 418 | −$55.875 | 7.9% | 0.062 | 0.027 | −0.179 |
| BOP ADX<20 | Lateral | 284 | −$35.715 | 8.5% | 0.107 | 0.001 | −0.280 |
| RSI reversal (ADX<20) | Lateral | 47 | −$4.707 | 8.5% | 0.174 | −0.086 | −2.01 |
| RSI trend (ADX>25) | Tendência | 174 | −$22.098 | 13.2% | 0.101 | 0.004 | −0.453 |
| Buy & Hold | — | 0 | −$296 | — | — | 0.003 | −1.59 |

### Análise

**Sem taxas, o filtro de regime transforma a estratégia:**

1. **BOP em lateralização (ADX < 20)**: PnL +$804, profit factor 1.08, Calmar 19.6 — o melhor BOP. O filtro remove os trades em tendência (que são perdedores para BOP neste mercado) e mantém os que funcionam.

2. **RSI reversal em lateralização**: a melhor estratégia de todas. Apenas 47 trades, win rate 76.6%, profit factor 1.66, Calmar 133, max drawdown 9%. A expectativa por trade é +$28.4 — cada trade adiciona 0.28% ao capital.

3. **BOP sem filtro**: PnL −$3.649 — perdedor porque metade dos trades acontece em tendência, onde BOP é contraproducente.

4. **BOP em tendência (ADX > 25)**: PnL −$2.238 — pior que sem filtro. BOP é momentum intrabar, não previsão de tendência; filtrar por tendência não ajuda.

> **A descoberta chave: este mercado é anti-persistente em escala longa** (VR < 1 em τ=64–512). Estratégias de reversão funcionam melhor que trend-following. O filtro ADX < 20 captura exatamente os períodos laterais onde a reversão domina.

### Com taxas, tudo é destruído

O RSI reversal tem apenas 47 trades — muito abaixo das 866 do BOP base — mas mesmo assim as taxas ($6.041) superam o PnL bruto ($1.334). O problema estrutural é o mesmo do exemplo 08: o edge bruto não é grande o suficiente para superar o custo de execução.

> **Mas a lição de regime sobrevive**: as taxas destroem o PnL, mas o **ranking** de estratégias é o mesmo. RSI reversal em lateralização é melhor que BOP em tendência. Se você adicionar o RSI reversal como **filtro de entrada** ao GBDT (exemplo 06) — que já superou taxas — o regime filtro pode melhorar o modelo.

---

## Passo 5 —Observações sobre Variance Ratio

A tabela de VR por bloco temporal (Passo 1.3) revela algo importante: **o regime muda não apenas entre períodos, mas também entre escalas**:

- Em τ=4 (1 hora), o mercado alterna entre VR > 1 (bloco 3, tendência) e VR < 1 (bloco 4, lateralização) — regimes transitórios.
- Em τ=512 (~5 dias), VR = 0.29 consistentemente — **reversão forte em todas as janelas**.

Isso significa que estratégias de ** Curta duração (~1h) precisam detectar regime em tempo real (ADX, Bollinger width). Estratégias de **longa duração (>1 dia) podem assumir reversão**.

> **Curtose nega** (platicúrtica em τ=512) significa que a distribuição de retornos de longo prazo é mais "achatada" que a normal — menos valores extremos. O mercado converte extremos em médias. Isso favorece mean reversion.

---

## Quando usar cada estratégia

| Regime | ADX | VR | Estratégia | Indicador |
|---|---|---|---|---|
| Tendência forte | > 25 | > 1 | Trend-following | BOP direcional, RSI > 50 |
| Lateralização | < 20 | < 1 | Mean reversion | RSI extremos (< 30 / > 70) |
| Transição | 20–25 | ≈ 1 | Não operar | — |

> **Filtro de regime não adiciona edge — ele remove anti-edge.** Em vez de tentar acertar mais, você tenta não operar nos momentos em que sua estratégia sistematicamente perde.

---

## Próximos passos

- **Regime + GBDT**: adicione ADX e BB width como features no pipeline do exemplo 06 — o modelo aprende a ajustar suas predições por regime automaticamente.
- **Fork da doutrina**: use ADX < 20 como filtro do gestor adaptativo — só entra quando o mercado está lateral e a reversão domina — veja [Fork da Doutrina](./11-fork-doutrina.pt.md).
- **Esteira ML completa**: automatize a detecção de regime num nó de ML (classificador de regime como pre-step do modelo) — veja [Esteira ML Completa](./10-esteira-ml-completa.pt.md).
- **Microestrutura + regime**: combine TFI/OBI (exemplo 08) com filtro ADX — entra apenas quando microestrutura e regime convergem.

---

> Voltar para: [README](../README.md) · [Microestrutura TFI](./08-tfi-backtest.pt.md) · [Modelo GBDT](./06-modelo-gbdt.pt.md) · [Esteira ML Completa](./10-esteira-ml-completa.pt.md)

_Last updated: 2026-08-12_
