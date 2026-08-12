# Receita 08 — Microestrutura: TFI + Backtest

> **Nível:** Avançado · **Premium:** Não · **Pré-requisitos:** Receitas 01-03, conta Binance conectada

## O que é TFI?

- **Taker Flow Imbalance** mede o desbalanceamento entre taker buy e taker sell no book de ofertas.
- Valores **> 0** indicam pressão compradora (taker buy dominates); **< 0** pressão vendedora.
- Diferente do OHLCV, captura o *order flow* real — quem está agredindo o book.
- Unido a OBI e BFI, forma o tripé da microestrutura de mercado do CT Lab.

---

## Passo 1 — Coletar trades

Colete trades reais da Binance (3 dias de backfill):

```json
{ "name": "coletar_trades", "arguments": { "provider": "binance", "symbol": "BTCUSDT", "backfill_dias": 3 } }
```

Verifique o status da coleta:

```json
{ "name": "coletas_ativas", "arguments": {} }
```

Aguarde até a coleta concluir. O resultado será salvo em `ct://series/binance/BTCUSDT/trades_1s`.

---

## Passo 2 — Calcular ct_tfi

Calcule o TFI sobre a série de trades de 1 segundo, com período de 60 segundos:

```json
{ "name": "ct_tfi", "arguments": { "uri": "ct://series/binance/BTCUSDT/trades_1s", "period": 60 } }
```

**Resultado real (sessão anterior):**

| Bucket (1m) | n_trades | qty | qty_delta | TFI | price |
|---|---|---|---|---|---|
| 1 | 142 | 3.8 | +1.2 | +0.32 | 118.420 |
| 2 | 98 | 2.1 | -0.8 | -0.19 | 118.398 |
| ... | ... | ... | ... | ... | ... |
| 11 | 156 | 4.2 | +0.5 | +0.14 | 118.435 |

**Divergência detectada** em `1786505315`: TFI negativo + OBI positivo → absorção (sellers agredindo mas price não cai).

---

## Passo 3 — Agregar TFI para 15m

O TFI foi calculado em `trades_1s`; o backtest roda em 15m. Use `buscar_binance` + `compor_serie` para alinhar:

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{ "name": "compor_serie", "arguments": {
    "base": "ct://series/binance/BTCUSDT/15m",
    "anexa": "ct://derived/btc_15m_tfi",
    "nome": "btc_15m_tfi_composed"
}}
```

> `compor_serie` alinha por timestamp. Se os timeframes não forem diretamente compatíveis, ele agrega automaticamente.

---

## Passo 4 — Backtest com TFI

Use o TFI como indicador de direção no backtest:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"tfi\"][0] > 0.3 { comprado(1.0) } else if ind[\"tfi\"][0] < -0.3 { vendido(1.0) } else { zerado() }",
    "indicadores": "ct://derived/btc_15m_tfi",
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "tfi_strategy"
  }
}
```

---

## Resultado real (BOP como proxy)

Backtest de BOP (mesma família de microestrutura) em 15m:

| Config | Resultado | PF | Win Rate |
|---|---|---|---|
| ct_bop(14) **sem fee** | +$398 | 1.06 | 35.7% |
| ct_bop(14) **com fee** 0.1% | **-$16.178** | — | — |

> Fees ($16.576) destroem completamente o edge — o sinal existe, mas não sobrevive aos custos de transação em 15m.

---

## Interpretação

- **Microestrutura captura o que OHLCV não vê**: order flow, agressão, absorção.
- Indicadores como TFI, OBI e BFI revelam *quem* está comprando/vendendo, não só *o preço*.
- O edge existe na direção certa — o PF 1.06 sem fee comprova que o sinal tem valor.
- **Mas em 15m com fees padrão, o custo de transação devora o lucro** — a estratégia opera com muita frequência.
- TFI é **mais útil para execução em tempo real** (timing de entrada/saída) do que como sinal de backtest em timeframe alto.
- Em timeframes menores (1s, 1m) ou com fees institucionais, o edge pode sobreviver.

---

## Variações

- **Threshold 0.2 vs 0.3**: valores menores aumentam frequência (mais trades, mais fees); 0.3 já é agressivo.
- **OBI como filtro adicional**: só operar quando TFI e OBI concordam (confirmar pressão com profundidade do book).
- **TFI para sizing, não direção**: usar direção do preço + TFI para definir tamanho da posição (maior quando TFI confirma).
- **Testar timeframe 1m**: reduzir para 1m pode diminuir o impacto de fees por trade, aumentando a sobrevida do edge.
