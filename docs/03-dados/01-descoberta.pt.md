# 01 · Descoberta de Ativos

Antes de ingerir candles, você precisa saber **quais ativos** vale a pena analisar. O CT Lab oferece duas ferramentas complementares de descoberta:

1. **`top_ativos`** — Rankings instantâneos: maiores altas, maiores baixas, maiores volumes.
2. **`filtrar_ativos`** — Screener paramétrico: filtre por preço mínimo, volume mínimo, etc.

Além disso, você pode **persistir** filtros nomeados com `salvar_filtro`, listá-los com `listar_filtros`, e removê-los com `excluir_filtro`.

---

## `top_ativos` — Rankings de Mercado

Retorna rankings de ativos por fonte (Binance, Yahoo, etc.) e janela temporal.

### Prompt de Chat IA

> "Quero ver os 10 ativos da Binance com maior alta hoje (janela diária)."

### Chamada de Ferramenta (snake_case)

```json
{
  "tool": "top_ativos",
  "arguments": {
    "fonte": "binance",
    "janela": "1d"
  }
}
```

### Retorno Esperado

```json
{
  "fonte": "binance",
  "janela": "1d",
  "alta": [
    { "symbol": "PEPEUSDT", "price": 0.00001234, "change_pct": 18.45 },
    { "symbol": "WIFUSDT",  "price": 2.34,        "change_pct": 15.20 },
    { "symbol": "FLOKIUSDT","price": 0.000234,    "change_pct": 12.11 }
  ],
  "menos_alta": [
    { "symbol": "DOGEUSDT", "price": 0.12, "change_pct": -8.3 }
  ],
  "baixa": [
    { "symbol": "BTCUSDT", "price": 64000, "change_pct": -1.2 }
  ],
  "menos_baixa": [
    { "symbol": "BNBUSDT", "price": 580, "change_pct": 0.3 }
  ],
  "volume": [
    { "symbol": "BTCUSDT",  "price": 64000, "volume": 1200000000 },
    { "symbol": "ETHUSDT",  "price": 3200,  "volume": 800000000 }
  ],
  "menos_volume": [
    { "symbol": "SCRLUSDT", "price": 0.01, "volume": 100000 }
  ],
  "atualizado_em": "2026-08-11T15:00:00Z"
}
```

### Campos do Retorno

| Campo | Descrição |
|-------|-----------|
| `alta` | Ativos com maior % de alta no período |
| `menos_alta` | Ativos com menor % de alta (próximos de zero ou negativos leves) |
| `baixa` | Ativos com maior % de baixa no período |
| `menos_baixa` | Ativos com menor % de baixa |
| `volume` | Ativos com maior volume negociado |
| `menos_volume` | Ativos com menor volume negociado |

### Parâmetros

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `fonte` | `"binance"` \| `"yahoo"` | Provedor de dados |
| `janela` | `"1d"`, `"4h"`, `"1h"`, ... | Janela temporal do ranking |

---

## `filtrar_ativos` — Screener Paramétrico

Filtra ativos por critérios quantitativos. Ideal para construir watchlists dinâmicas.

### Prompt de Chat IA

> "Filtre ativos da Binance com preço mínimo de $0.50 e volume mínimo de $10 milhões."

### Chamada de Ferramenta

```json
{
  "tool": "filtrar_ativos",
  "arguments": {
    "provider": "binance",
    "min_price": 0.50,
    "min_volume": 10000000
  }
}
```

### Retorno Esperado

```json
{
  "provider": "binance",
  "criterios": {
    "min_price": 0.50,
    "min_volume": 10000000
  },
  "ativos": [
    { "symbol": "BTCUSDT",  "price": 64000.0,   "volume": 1200000000 },
    { "symbol": "ETHUSDT",  "price": 3200.0,    "volume":  800000000 },
    { "symbol": "SOLUSDT",  "price": 145.0,     "volume":  450000000 }
  ],
  "total": 3
}
```

### Parâmetros do Screener

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `provider` | `string` | Fonte dos dados (ex.: `"binance"`) |
| `min_price` | `number` | Preço mínimo do ativo |
| `max_price` | `number` | Preço máximo do ativo |
| `min_volume` | `number` | Volume mínimo em USD |
| `max_volume` | `number` | Volume máximo em USD |
| `min_change_pct` | `number` | Variação percentual mínima |
| `max_change_pct` | `number` | Variação percentual máxima |

---

## `salvar_filtro` — Persistir um Screener Nomeado

Repetir os mesmos critérios toda hora é tedioso. Salve o filtro e reutilize-o.

### Prompt de Chat IA

> "Salve o filtro atual com o nome 'altcap_high_volume'."

### Chamada de Ferramenta

```json
{
  "tool": "salvar_filtro",
  "arguments": {
    "name": "altcap_high_volume",
    "config": {
      "provider": "binance",
      "min_price": 0.50,
      "min_volume": 10000000
    }
  }
}
```

### Retorno Esperado

```json
{
  "name": "altcap_high_volume",
  "salvo": true
}
```

---

## `listar_filtros` — Listar Screeners Salvos

### Prompt de Chat IA

> "Liste todos os filtros que salvei."

### Chamada de Ferramenta

```json
{
  "tool": "listar_filtros",
  "arguments": {}
}
```

### Retorno Esperado

```json
{
  "filtros": [
    {
      "name": "altcap_high_volume",
      "config": {
        "provider": "binance",
        "min_price": 0.50,
        "min_volume": 10000000
      },
      "criado_em": "2026-08-10T12:00:00Z"
    },
    {
      "name": "pennies_com_volume",
      "config": {
        "provider": "binance",
        "max_price": 0.10,
        "min_volume": 5000000
      },
      "criado_em": "2026-08-09T08:30:00Z"
    }
  ]
}
```

---

## `excluir_filtro` — Remover um Screener

### Prompt de Chat IA

> "Apague o filtro 'pennies_com_volume'."

### Chamada de Ferramenta

```json
{
  "tool": "excluir_filtro",
  "arguments": { "name": "pennies_com_volume" }
}
```

### Retorno Esperado

```json
{
  "excluido": true
}
```

---

## Workflow Recomendado

```
top_ativos  ──>  identificação de oportunidades
     │
     ▼
filtrar_ativos  ──>  refinamento por critérios quantitativos
     │
     ▼
salvar_filtro  ──>  persistência para reuso diário
     │
     ▼
listar_filtros  ──>  auditoria dos filtros salvos
```

1. Use `top_ativos` para uma **visão rápida** do mercado.
2. Use `filtrar_ativos` para **refinar** por critérios objetivos.
3. Salve filtros úteis com `salvar_filtro` para não reescrever critérios.
4. Periodicamente, audite com `listar_filtros` e limpe com `excluir_filtro`.

---

## Exemplo em TypeScript (SDK)

```typescript
// Rankings instantâneos
const topBinance = await Ct.topAtivos({
  fonte: "binance",
  janela: "1d",
});
console.log("Maior alta:", topBinance.alta[0]);

// Screener paramétrico
const altCoins = await Ct.filtrarAtivos({
  provider: "binance",
  minPrice: 0.50,
  minVolume: 10_000_000,
});
console.log(`${altCoins.total} ativos correspondem`);

// Persistir para reuso
await Ct.salvarFiltro({
  name: "altcap_high_volume",
  config: {
    provider: "binance",
    minPrice: 0.50,
    minVolume: 10_000_000,
  },
});
```

---

[← Voltar para a categoria 03](README.pt.md)
