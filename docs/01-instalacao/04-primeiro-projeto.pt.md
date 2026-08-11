# 04 — Primeiro Projeto

Parabéns! Você chegou ao último passo da instalação. Agora vamos fazer um
**percurso completo de ponta a ponta**: buscar dados de mercado, calcular um
indicador, executar um backtest e interpretar os resultados.

Este documento é prático — cada seção mostra o que você digita no chat do CT
Lab e quais ferramentas a IA chama por trás dos panos.

---

## Sumário

- [Visão geral do projeto](#visão-geral-do-projeto)
- [Passo 1 — Buscar dados (BTCUSDT 15m)](#passo-1--buscar-dados-btcusdt-15m)
- [Passo 2 — Calcular RSI](#passo-2--calcular-rsi)
- [Passo 3 — Backtest simples](#passo-3--backtest-simples)
- [Passo 4 — Interpretar resultados](#passo-4--interpretar-resultados)
- [Bonus: usando Python](#bonus-usando-python)

---

## Visão geral do projeto

Nossa estratégia de exemplo será simples:

> **"Se o preço de fechamento de hoje for maior que o de ontem, comprar.
> Caso contrário, ficar zerado."**

Para isso, precisaremos de:

1. **Dados de mercado** — candles de BTCUSDT em 15m da Binance.
2. **Indicador** — RSI de 14 períodos para acompanhar o momentum.
3. **Backtest** — simular a estratégia com capital de $1.000.

---

## Passo 1 — Buscar dados (BTCUSDT 15m)

### No chat do CT Lab, digite:

> **"Busque a série BTCUSDT em intervalo de 15m na Binance."**

### O que a IA faz por trás:

A IA chama a ferramenta `buscar_serie` (JSON-RPC) / `buscarSerie` (TypeScript SDK),
que internamente usa `buscar_binance` / `buscarBinance`:

**JSON-RPC (MCP):**
```json
{
  "method": "tools/call",
  "params": {
    "name": "buscar_serie",
    "arguments": {
      "provider": "binance",
      "symbol": "BTCUSDT",
      "interval": "15m"
    }
  }
}
```

**TypeScript SDK:**
```typescript
const result = await Ct.buscarSerie({
  provider: "binance",
  symbol: "BTCUSDT",
  interval: "15m"
});
```

### Resposta esperada:

```text
🤖 Série carregada com sucesso!

   • URI: ct://series/binance/BTCUSDT/15m
   • Candles carregados: 500
   • Primeiro candle: 2026-08-06 00:00 UTC
   • Último candle: 2026-08-11 15:45 UTC
```

A **URI** (`ct://series/binance/BTCUSDT/15m`) é o identificador único da
série. Você a usará nas próximas etapas para referenciar os dados.

> 💡 A URI é permanente: mesmo que você feche o CT Lab, a série fica no cache
> e pode ser reutilizada.

---

## Passo 2 — Calcular RSI

### No chat do CT Lab, digite:

> **"Calcule o RSI de 14 períodos para essa série."**

### O que a IA faz por trás:

A IA usa a URI da série anterior e chama a ferramenta `rsi`:

**JSON-RPC (MCP):**
```json
{
  "method": "tools/call",
  "params": {
    "name": "rsi",
    "arguments": {
      "uri": "ct://series/binance/BTCUSDT/15m",
      "period": 14
    }
  }
}
```

**TypeScript SDK:**
```typescript
const result = await Ct.rsi({
  uri: "ct://series/binance/BTCUSDT/15m",
  period: 14
});
```

### Resposta esperada:

```text
🤖 RSI(14) calculado para BTCUSDT 15m.

   • URI do indicador: ct://derived/binance/BTCUSDT/15m/rsi_14
   • Valores: ["rsi"]
   • Último valor: 58.32
   • Período válido a partir de: 2026-08-06 03:30 UTC
```

O RSI varia de 0 a 100:

- **> 70**: mercado sobrecomprado (possível reversão de baixa)
- **< 30**: mercado sobrevendido (possível reversão de alta)
- **30–70**: zona neutra

> 📊 O indicador também fica em cache. Se você pedi-lo de novo mais tarde,
> a resposta será instantânea.

---

## Passo 3 — Backtest simples

### No chat do CT Lab, digite:

> **"Rode um backtest com a seguinte estratégia: se o fechamento atual for
> maior que o anterior, ficar comprado; senão, zerar. Use $1.000 de capital
> e 0.1% de taxa por trade."**

### O que a IA faz por trás:

A IA monta o script Rhai e chama a ferramenta `ct_backtest` (JSON-RPC) /
`ctBacktest` (TypeScript SDK):

**JSON-RPC (MCP):**
```json
{
  "method": "tools/call",
  "params": {
    "name": "ct_backtest",
    "arguments": {
      "serie": "ct://series/binance/BTCUSDT/15m",
      "estrategia_script": "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
      "capital_inicial": 1000,
      "fee_pct": 0.001,
      "nome": "exemplo_momentum"
    }
  }
}
```

**TypeScript SDK:**
```typescript
const result = await Ct.ctBacktest({
  serie: "ct://series/binance/BTCUSDT/15m",
  estrategiaScript: "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
  capitalInicial: 1000,
  feePct: 0.001,
  nome: "exemplo_momentum"
});
```

### Sintaxe Rhai para backtests

A estratégia é escrita em **Rhai**, uma linguagem de scripting segura e
embutida no ct-mcp-server:

| Função | Significado |
|--------|-------------|
| `comprado(1.0)` | Ficar **100% comprado** (long). O valor deve ser float. |
| `vendido(1.0)` | Ficar **100% vendido** (short). O valor deve ser float. |
| `zerado()` | Ficar **100% zerado** (flat / sem posição). |
| `close[0]` | Preço de fechamento do candle **atual**. |
| `close[1]` | Preço de fechamento do candle **anterior**. |
| `ind["alias"][0]` | Valor atual de um indicador com alias. |

> ⚠️ **Importante**: sempre use `float` nos argumentos: `comprado(1.0)`, não
> `comprado(1)`. O Rhai é tipado e exige f64.

---

## Passo 4 — Interpretar resultados

### Resposta esperada:

```text
🤖 Backtest concluído — "exemplo_momentum"

   📈 Métricas de desempenho:
   ┌────────────────────────────────────────────────┐
   │  Retorno total:          +12.34%                │
   │  PnL total:              +$123.40               │
   │  PnL bruto:              +$145.20               │
   │  Fees totais:            -$21.80                │
   │  Número de trades:       87                     │
   │  Win rate:               54.0%                  │
   │  Profit factor:          1.32                   │
   │  Drawdown máximo:        -8.45%                 │
   │  Sharpe ratio:           1.21                   │
   │  Sortino ratio:          1.68                   │
   │  Calmar ratio:           1.46                   │
   └────────────────────────────────────────────────┘

   • Equity URI: ct://backtest/exemplo_momentum/equity
   • Trades URI: ct://backtest/exemplo_momentum/trades
```

### Guia de interpretação

| Métrica | O que significa | Valor de referência |
|---------|-----------------|---------------------|
| **Retorno total** | Ganho/perda percentual no período | > 0 é bom |
| **Win rate** | % de trades lucrativos | > 50% é razoável |
| **Profit factor** | Ganhos ÷ perdas | > 1.0 é lucrativo |
| **Sharpe** | Retorno ajustado ao risco | > 1.0 é bom, > 2.0 é ótimo |
| **Sortino** | Retorno ajustado à volatilidade negativa | Maior que Sharpe = bom |
| **Calmar** | Retorno ÷ drawdown máximo | > 1.0 é desejável |
| **Drawdown máximo** | Maior queda do patrimônio | Menor (em valor absoluto) é melhor |
| **Fees totais** | Custos de transação | Depende do número de trades |

> 💡 Esta estratégia é **educacional** e não otimizada. Em um projeto real,
> você combinaria múltiplos indicadores (RSI + SMA, por exemplo) para filtrar
> falsos sinais.

---

## Bonus: usando Python

Se você prefere scripting diretamente, pode usar o CT Lab via Python com `uv`:

```bash
# Instalar dependências
uv pip install ct-mcp-client

# Criar o script do primeiro projeto
cat > primeiro_projeto.py << 'EOF'
import asyncio
from ct_mcp_client import Client

async def main():
    client = Client()

    # Passo 1: buscar série
    serie = await client.call("buscar_serie", {
        "provider": "binance",
        "symbol": "BTCUSDT",
        "interval": "15m"
    })
    print(f"Série: {serie['uri']} ({serie['row_count']} candles)")

    # Passo 2: calcular RSI
    rsi = await client.call("rsi", {
        "uri": serie["uri"],
        "period": 14
    })
    print(f"RSI atual: {rsi['latest'][0]:.2f}")

    # Passo 3: backtest
    bt = await client.call("ct_backtest", {
        "serie": serie["uri"],
        "estrategia_script": "if close[0] > close[1] { comprado(1.0) } else { zerado() }",
        "capital_inicial": 1000,
        "fee_pct": 0.001,
        "nome": "exemplo_momentum"
    })
    print(f"Retorno: {bt['retorno_total']:.2%}")
    print(f"Sharpe:  {bt['sharpe']:.2f}")

asyncio.run(main())
EOF

# Rodar
uv run primeiro_projeto.py
```

> A API Python usa `snake_case` para nomes de ferramentas, igual ao JSON-RPC.

---

## Próximos Passos

Agora você tem tudo configurado e funcionando! Explore mais:

| Tópico | Como explorar |
|--------|---------------|
| **Mais indicadores** | Peça: "Calcule SMA, MACD e Bollinger para BTCUSDT." |
| **Estratégias complexas** | Combine indicadores no script Rhai usando `ind["alias"][0]`. |
| **Múltiplas séries** | Busque ETHUSDT, SOLUSDT e compare. |
| **Otimização** | Use `otimizar_hiperparametros` para encontrar os melhores parâmetros. |
| **ML** | Se você tem Premium, explore `montar_esteira_ml`. |

- ⬅️ **[03 — Conexão MCP](./03-conexao-mcp)** — Voltar
- ⬅️ **[Índice de Instalação](./README)**

---

> 🎉 **Pronto!** Você completou a instalação do CT Lab e rodou seu primeiro
> projeto de ponta a ponta. Bons trades!

_Last updated: 2026-08-11_
