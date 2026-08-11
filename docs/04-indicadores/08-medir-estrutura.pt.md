# Medir Estrutura — `ct_medir_estrutura`

> **Premium.** Mede variance ratio e curtose por escala e regime de volatilidade **antes** de operar. Responde: "há estrutura para colher neste ativo?"

Antes de construir qualquer estratégia, é fundamental saber se o ativo tem **estrutura** — isto é, se o caminho dos preitos tem propriedades exploráveis (mean-reversion, momentum, etc.). `ct_medir_estrutura` responde isso com métricas rigorosas.

---

## O que mede

### Variance Ratio (VR)

`VR(τ) = Var(r_τ) / (τ · Var(r_1))`

| VR | Interpretação |
|---|---|
| VR ≈ 1 | Caminho aleatório (martingale) — **sem estrutura** |
| VR < 1 | Anti-persistência (mean-reversion) — oscila, reverte |
| VR > 1 | Persistência (momentum) — tendência, caminha |

### Curtose

Mede a "cauda" da distribuição de retornos. Curtose alta = eventos extremos raros mas grandes (crashes, rebounds).

---

## Chamada

```json
{
  "name": "ct_medir_estrutura",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m"
  }
}
```

**Retorno (formato resumido):**
```json
{
  "uri": "ct://series/binance/BTCUSDT/15m",
  "blocos": 8,
  "escalas": [1, 5, 15, 60, 240],
  "vr_por_escala": {
    "1": [0.85, 0.72, 0.91, 0.68, 0.79, 0.88, 0.75, 0.82],
    "5": [0.71, 0.59, 0.82, 0.55, 0.68, 0.77, 0.61, 0.70],
    "15": [0.65, 0.51, 0.78, 0.48, 0.62, 0.71, 0.54, 0.63],
    "60": [0.58, 0.45, 0.72, 0.42, 0.55, 0.64, 0.48, 0.57],
    "240": [0.52, 0.40, 0.68, 0.38, 0.50, 0.59, 0.43, 0.51]
  },
  "curtose_por_escala": {
    "1": 72.3,
    "5": 45.1,
    "15": 28.7,
    "60": 15.2,
    "240": 8.4
  },
  "regimes_vol": ["baixa", "media", "alta"],
  "vr_por_regime": {
    "baixa": { "vr_1": 1.5, "vr_16": 1.8, "vr_64": 2.0 },
    "media": { "vr_1": 0.9, "vr_16": 0.7, "vr_64": 0.6 },
    "alta": { "vr_1": 0.71, "vr_16": 0.59, "vr_64": 0.49 }
  },
  "veredito": {
    "vol_alta": "anti-persistente_estavel (VR<1 em 8/8 blocos)",
    "vol_baixa": "persistente (VR 1.2-2.0)",
    "curtose": "altissima (15m tau=1: 72) — reversões grandes e raras"
  }
}
```

---

## Como interpretar

### Exemplo: BTCUSDT 15m

| Regime de vol | VR(τ=1) | VR(τ=16) | VR(τ=64) | Interpretação |
|---|---|---|---|---|
| **Vol alta** | 0.71 | 0.59 | 0.49 | Anti-persistente estável — reverte. 8/8 blocos VR<1 |
| **Vol baixa** | 1.5 | 1.8 | 2.0 | Persistente — caminha. Mean-reversion perde |

**Conclusão:** Em vol alta, o BTCWT 15m oscila e reverte (VR<1). Em vol baixa, ele anda (VR>1). A estrutura existe, mas é **dependente do regime**.

### Regras de bolso

| Situação | VR | Ação |
|---|---|---|
| VR ≈ 1 em todos os regimes | Sem estrutura | Não há o que colher — procure outro ativo |
| VR < 1 (anti-persistência) | Mean-reversion | Estratégias de reversão podem funcionar |
| VR > 1 (persistência) | Momentum | Estratégias de tendência podem funcionar |
| Curtose muito alta | Caudas gordas | Stops são essenciais — eventos extremos acontecem |

---

## Por que isto importa

A doutrina CT estabelece: **num caminho sem estrutura (martingale), E[P&L] = 0 para qualquer política**. Antes de otimizar qualquer estratégia, meça a estrutura. Se não há estrutura, nenhum indicador vai gerar edge — é auto-engano.

> Veja também: [Doutrina CT — Axioma Matemático](../08-doutrina/)

---

## Reprodução

O exemplo acima é reproduzível com:
- Série: BTCUSDT 15m Binance, 2017-09 → 2026-07
- 8 blocos temporais
- 4 timeframes (5m, 15m, 1h, 4h) confirmam o padrão

Dados de: `examples/estudar_estrutura.rs` (cache local) e tool `ct_medir_estrutura` (qualquer série do store).

---

> Próximo: [Indicadores custom](./09-indicadores-custom.pt.md) · [Cookbook](./07-cookbook.pt.md)
