# Planos Free vs Premium

> **Pasta:** `docs/02-conceitos/06-free-vs-premium.pt.md`  
> **Leitura relacionada:** [`01-visao-geral`](./01-visao-geral.pt.md) ·
> [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md)  
> **Público-alvo:** todos os públicos

---

## Visão Geral

O CT Lab opera com dois planos: **Free** (gratuito) e **Premium** (pago). A
diferença entre eles não é apenas quantidade de recursos — é também o tipo de
capacidade analítica disponível. O plano Free cobre análise básica de dados e
backtest; o Premium desbloqueia o arsenal completo de indicadores proprietários
CT, ML, microestrutura e testes de robustez.

O licenciamento é **local** — validado pela URI `ct://license/info` no
`ct-labd`.

---

## Tabela de Comparação

| Recurso | Free | Premium |
|---------|------|---------|
| **Cache de séries** | 1 série | 100 séries |
| **Indicadores públicos** (36) | ✅ SMA, EMA, RSI, MACD, Bollinger, ATR, … | ✅ Todos os 36 |
| **Indicadores CT** (17) | ❌ | ✅ ct_bop, ct_tfi, ct_bfi, ct_obi, ct_candle, ct_swing, ct_range, ct_tendencia, ct_fibo_candle, ct_candle_classificado, ct_momento, bop, tfi, bfi, obi, dbi_01, dbi_1, mpo |
| **Backtest básico** | ✅ `ct_backtest` | ✅ |
| **Buscar backtests** | ✅ `ct_buscar_backtests` | ✅ |
| **ML pipeline** | ❌ | ✅ `montar_esteira_ml`, `aplicar_modelo`, `otimizar_hiperparametros` |
| **Microestrutura** | ❌ | ✅ `coletar_book`, `coletar_trades`, `consultar_book`, `consultar_trades` |
| **Coleção de klines** | ✅ `coletar_klines`, `parar_klines` | ✅ |
| **Teste de sobrevivência** | ❌ | ✅ `ct_testar_sobrevivencia` |
| **Bibliotecas (libs)** | ❌ | ✅ `salvar_lib`, `ler_lib`, `listar_libs`, `excluir_lib` |
| **Filtros** | ❌ | ✅ `salvar_filtro`, `listar_filtros`, `excluir_filtro` |
| **Modelos ML** | ❌ | ✅ `listar_modelos`, `excluir_modelo` |
| **Prompts públicos** | ✅ `saudacao`, `comecar`, `backtest` | ✅ |
| **Prompts premium** | ❌ | ✅ `coleta`, `esteira`, `fundacao`, `regime` |
| **Discovery** | ✅ `buscar_serie`, `listar_series`, `top_ativos` | ✅ |
| **Importação CSV** | ✅ `importar_csv` | ✅ |
| **Billing** | ✅ `comprar_premium`, `cancelar_assinatura` | ✅ |

---

## Os 17 Indicadores CT (Premium)

Estes são os indicadores proprietários exclusivos do plano Premium:

| # | Tool (MCP) | SDK (TS) | Categoria |
|---|-----------|----------|-----------|
| 1 | `ct_bop` | `ctBop` | Balance of Power |
| 2 | `ct_tfi` | `ctTfi` | Trend Flow Index |
| 3 | `ct_bfi` | `ctBfi` | Buying Force Index |
| 4 | `ct_obi` | `ctObi` | Order Book Imbalance |
| 5 | `ct_candle` | `ctCandle` | Candle Analysis |
| 6 | `ct_swing` | `ctSwing` | Swing Analysis |
| 7 | `ct_range` | `ctRange` | Range Analysis |
| 8 | `ct_tendencia` | `ctTendencia` | Trend Analysis |
| 9 | `ct_fibo_candle` | `ctFiboCandle` | Fibonacci Candle |
| 10 | `ct_candle_classificado` | `ctCandleClassificado` | Classified Candle |
| 11 | `ct_momento` | `ctMomento` | Momentum Analysis |
| 12 | `bop` | `bop` | Balance of Power (legacy) |
| 13 | `tfi` | `tfi` | Trend Flow Index (legacy) |
| 14 | `bfi` | `bfi` | Buying Force Index (legacy) |
| 15 | `obi` | `obi` | Order Book Imbalance (legacy) |
| 16 | `dbi_01` | `dbi01` | Delta Balance Index 01 |
| 17 | `dbi_1` | `dbi1` | Delta Balance Index 1 |

> Os 36 indicadores públicos estão documentados em `docs/03-indicadores/`.

---

## Como Funciona o Gate de Licença

O gate de licença é aplicado no `ct-labd`. Quando a IA tenta invocar uma tool
Premium com uma licença Free, a chamada retorna um erro estruturado:

```json
{
  "error": "premium_required",
  "tool": "ct_bop",
  "message": "Esta ferramenta requer o plano Premium.",
  "upgrade_uri": "ct://license/upgrade"
}
```

O LLM, ao receber este erro, pode informar o usuário e sugerir o upgrade.

### Diagrama

```
  IA invoca tool Premium (ex: ct_bop)
       │
       ▼
  ct-mcp-server ──► ct-labd
       │                 │
       │          ┌──────┴──────┐
       │          ▼             ▼
       │    License OK     License Free/Não
       │    (Premium)      registrado
       │          │             │
       │          ▼             ▼
       │    Executa tool    Retorna erro
       │    normalmente     "premium_required"
       │          │             │
       └──────────┴─────────────┘
                       │
                       ▼
              LLM informa o usuário
```

---

## Como Verificar o Status da Licença

A URI `ct://license/info` retorna as informações da licença presente:

```python
# Verifica o status da licença
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import read_resource; "
    "info = read_resource('ct://license/info'); "
    "print(info)"
], capture_output=True, text=True)
print(result.stdout)
```

### Exemplos de resposta

**Plano Free:**
```json
{
  "plan": "free",
  "premium": false,
  "max_cache": 1,
  "indicators_public": 36,
  "indicators_ct": 0,
  "ml_enabled": false
}
```

**Plano Premium:**
```json
{
  "plan": "premium",
  "premium": true,
  "max_cache": 100,
  "indicators_public": 36,
  "indicators_ct": 17,
  "ml_enabled": true,
  "valid_until": "2026-12-31"
}
```

---

## Como Fazer Upgrade

### Usando a tool MCP `comprar_premium`

```typescript
// SDK TypeScript: inicia o processo de upgrade
const result = await Ct.comprarPremium({
    // parâmetros de billing conforme necessário
});
// → Retorna informações de pagamento/assinatura
```

### Usando Python

```python
# Inicia o processo de upgrade para Premium
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import comprar_premium; "
    "result = comprar_premium(); "
    "print(result)"
], capture_output=True, text=True)
print(result.stdout)
# → Retorna dados de checkout/assinatura
```

### Cancelar assinatura

```python
# Cancela a assinatura Premium
import subprocess
result = subprocess.run([
    "uv", "run", "python", "-c",
    "from ct_lab import cancelar_assinatura; "
    "result = cancelar_assinatura(); "
    "print(result)"
], capture_output=True, text=True)
print(result.stdout)
```

---

## Púbicanos: Visibilidade das Tools

O CT Lab classifica ferramentas em 3 níveis de visibilidade:

| Nível | Variável de ambiente | Descrição |
|-------|---------------------|-----------|
| **public** | (sempre visível) | 36 indicadores, dados, backtest, billing, prompts básicos |
| **private** | Licença Premium | 17 indicadores CT, ML, microestrutura, libs, prompts premium |
| **diag** | `CT_MCP_DIAG` | Ferramentas de desenvolvedor — não para usuários finais |

> As ferramentas `diag` só aparecem quando a variável de ambiente
> `CT_MCP_DIAG` está definida. Elas não devem ser usadas por usuários finais.

---

## FAQ

| Pergunta | Resposta |
|----------|----------|
| Preciso de internet para o plano Free? | Sim — para buscar dados dos providers. O cache é local. |
| O Premium expira? | Sim, conforme `valid_until` em `ct://license/info`. |
| Posso usar ML no plano Free? | Não. Todo o pipeline de ML é Premium. |
| Backtest é Free? | Sim, backtest básico (`ct_backtest`) é Free. |
| Quantas séries posso ter no cache? | Free: 1. Premium: 100. |

---

## Próximos Passos

- [`01-visao-geral`](./01-visao-geral.pt.md) — visão geral do ecossistema
- [`05-mcp-protocolo`](./05-mcp-protocolo.pt.md) — tools, resources, prompts
- `docs/03-indicadores/` — documentação dos 36 indicadores públicos
