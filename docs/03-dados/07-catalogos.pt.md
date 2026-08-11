# 07 · Catálogos ao Vivo

O CT Lab mantém **catálogos ao vivo** — resources especiais que retornam lista completa de componentes disponíveis em tempo real. Diferente de documentação estática, que envelhece, os catálogos **sempre refletem o estado atual** do sistema, incluindo componentes novos, licenciados e privados.

---

## Por Que Catálogos Importam?

Documentação tradicional tem um problema: **envelhece**. Um README que lista "5 indicadores disponíveis" fica desatualizado quando o czas 6º é adicionado. O usuário lê docs desatualizadas e perde recursos.

Os catálogos do CT Lab resolvem isto:

| Característica | Documentação Estática | Catálogos ao Vivo |
|---------------|:---------------------:|:-----------------:|
| Sempre atual | ❌ | ✅ |
| Inclui licenciados | ❌ | ✅ (se licenciado) |
| Inclui privados | ❌ | ✅ (se licenciado) |
| Versionado com o produto | ❌ | ✅ |
| Modelo IA lê direto | ❌ | ✅ |

> O modelo IA pode consultar `ct://indicators/catalog` a qualquer momento para saber quais indicadores estão disponíveis **agora**, sem depender de documentação que pode estar desatualizada.

---

## Os Quatro Catálogos

### 1. `ct://sources/catalog` — Fontes de Dados

Lista todos os provedores de dados disponíveis.

#### Prompt de Chat IA

> "Quais fontes de dados estão disponíveis no CT Lab?"

#### Resource URI

```
ct://sources/catalog
```

#### Resposta Esperada

```json
{
  "uri": "ct://sources/catalog",
  "sources": [
    {
      "provider": "binance",
      "name": "Binance",
      "description": "Exchange de criptomoedas",
      "supported_intervals": ["1m", "5m", "15m", "1h", "4h", "1d"],
      "features": ["spot", "futures"],
      "requires_api_key": false
    },
    {
      "provider": "yahoo",
      "name": "Yahoo Finance",
      "description": "Ações, ETFs, índices",
      "supported_intervals": ["1d", "1wk", "1mo"],
      "features": ["stocks", "etf", "index"],
      "requires_api_key": false
    }
  ]
}
```

### 2. `ct://indicators/catalog` — Indicadores Técnicos

Lista todos os indicadores técnicos disponíveis (públicos + privados se licenciado).

#### Prompt de Chat IA

> "Lista todos os indicadores técnicos disponíveis."

#### Resource URI

```
ct://indicators/catalog
```

#### Resposta Esperada

```json
{
  "uri": "ct://indicators/catalog",
  "indicators": [
    {
      "name": "rsi",
      "display_name": "Relative Strength Index",
      "category": "momentum",
      "params": { "period": { "type": "number", "default": 14, "min": 1 } },
      "private": false
    },
    {
      "name": "macd",
      "display_name": "MACD",
      "category": "momentum",
      "params": {
        "fast_period": { "type": "number", "default": 12 },
        "slow_period": { "type": "number", "default": 26 },
        "signal_period": { "type": "number", "default": 9 }
      },
      "private": false
    },
    {
      "name": "bollinger",
      "display_name": "Bollinger Bands",
      "category": "volatility",
      "params": {
        "period": { "type": "number", "default": 20 },
        "std_dev": { "type": "number", "default": 2.0 }
      },
      "private": false
    }
  ]
}
```

> Indicadores `private: true` só aparecem se você tiver a licença correspondente. Isto evita que o modelo IA tente usar um indicador que não está licenciado.

### 3. `ct://pipeline/catalog` — Operações de Pipeline

Lista as operações disponíveis para montar pipelines de processamento.

#### Prompt de Chat IA

> "Quais operações de pipeline posso usar?"

#### Resource URI

```
ct://pipeline/catalog
```

#### Resposta Esperada

```json
{
  "uri": "ct://pipeline/catalog",
  "operations": [
    {
      "name": "window",
      "description": "Janela deslizante para cálculo incremental",
      "params": { "size": { "type": "number", "required": true } }
    },
    {
      "name": "normalize",
      "description": "Normalização min-max ou z-score",
      "params": {
        "method": { "type": "string", "enum": ["minmax", "zscore"], "default": "minmax" }
      }
    },
    {
      "name": "lag",
      "description": "Desloca a série por N períodos",
      "params": { "periods": { "type": "number", "default": 1 } }
    }
  ]
}
```

### 4. `ct://ml/catalog` — Componentes de Machine Learning

Lista os componentes de ML disponíveis para montar esteiras e pipelines de aprendizado.

#### Prompt de Chat IA

> "Quais componentes de machine learning estão disponíveis?"

#### Resource URI

```
ct://ml/catalog
```

#### Resposta Esperada

```json
{
  "uri": "ct://ml/catalog",
  "components": [
    {
      "name": "linear_regression",
      "display_name": "Linear Regression",
      "category": "regression",
      "params": {
        "fit_intercept": { "type": "boolean", "default": true }
      },
      "private": false
    },
    {
      "name": "random_forest",
      "display_name": "Random Forest Regressor",
      "category": "regression",
      "params": {
        "n_estimators": { "type": "number", "default": 100 },
        "max_depth": { "type": "number", "default": null }
      },
      "private": false
    }
  ]
}
```

---

## Como o Modelo IA Usa os Catálogos

O fluxo típico do modelo IA:

```
1. Lê ct://indicators/catalog     → descobre que "rsi" existe
2. Confirma parâmetros              → period=14, etc.
3. Chama Ct.rsi({ uri, period: 14 })  → computa o indicador
4. Lê o resultado via resource      → analisa e interpreta
```

Sem o catálogo, o modelo teria que adivinhar nomes e parâmetros — o que levaria a erros. Com o catálogo, o modelo descobre **programaticamente** o que está disponível.

---

## Benefícios Resumidos

| Benefício | Explicação |
|-----------|------------|
| **Sempre atual** | Catálogos são gerados dinamicamente, nunca hardcoded |
| **Sem docs obsoletas** | Não depende de markdown sendo atualizado manualmente |
| **Descoberta automática** | Modelos IA descobrem novos componentes sem prompt humano |
| **Filtro por licença** | Componentes privados só aparecem para usuários licenciados |
| **Schema de parâmetros** | Cada componente lista seus parâmetros e tipos |

---

## Exemplo em TypeScript

```typescript
// Descobrir todos os indicadores disponíveis
const catalogResource = await Ct.readResource("ct://indicators/catalog");
const indicators = catalogResource.indicators;
console.log(`${indicators.length} indicadores disponíveis:`);
indicators.forEach(ind => {
  const visibility = ind.private ? "🔒 privado" : "🌐 público";
  console.log(`  ${ind.display_name} (${ind.name}) — ${ind.category} ${visibility}`);
});

// Verificar fontes de dados
const sourcesCatalog = await Ct.readResource("ct://sources/catalog");
sourcesCatalog.sources.forEach(src => {
  console.log(`  ${src.name}: ${src.supported_intervals.join(", ")}`);
});
```

---

## Exemplo em Python com `uv`

```python
# uv add ct-mcp-client

import asyncio
from ct_mcp_client import CtLabClient

async def main():
    client = CtLabClient()

    # Listar indicadores disponíveis
    catalog = await client.read_resource("ct://indicators/catalog")
    print("Indicadores disponíveis:")
    for ind in catalog["indicators"]:
        LOCK = "🔒" if ind.get("private") else "🌐"
        print(f"  {LOCK} {ind['display_name']} ({ind['name']})")

    # Verificar pipeline ops
    ops = await client.read_resource("ct://pipeline/catalog")
    print(f"\nOperações de pipeline: {len(ops['operations'])}")
    for op in ops["operations"]:
        print(f"  - {op['name']}: {op['description']}")

asyncio.run(main())
```

```bash
uv run main.py
```

---

[← Voltar para a categoria 03](README.pt.md)
