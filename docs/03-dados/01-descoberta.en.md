# 01 · Asset Discovery

Before ingesting candles, you need to know **which assets** are worth analyzing. CT Lab offers two complementary discovery tools:

1. **`top_ativos`** — Instant rankings: top gainers, top losers, top volume.
2. **`filtrar_ativos`** — Parametric screener: filter by minimum price, minimum volume, etc.

Additionally, you can **persist** named screeners with `salvar_filtro`, list them with `listar_filtros`, and delete them with `excluir_filtro`.

---

## `top_ativos` — Market Rankings

Returns asset rankings by source (Binance, Yahoo, etc.) and time window.

### AI Chat Prompt

> "Show me the top 10 Binance assets with the biggest gains today (daily window)."

### Tool Call (snake_case)

```json
{
  "tool": "top_ativos",
  "arguments": {
    "fonte": "binance",
    "janela": "1d"
  }
}
```

### Expected Return

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

### Return Fields

| Field | Description |
|-------|-------------|
| `alta` | Assets with the biggest % gain in the period |
| `menos_alta` | Assets with the smallest % gain (near zero or slight negatives) |
| `baixa` | Assets with the biggest % loss in the period |
| `menos_baixa` | Assets with the smallest % loss |
| `volume` | Assets with the highest trading volume |
| `menos_volume` | Assets with the lowest trading volume |

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `fonte` | `"binance"` \| `"yahoo"` | Data provider |
| `janela` | `"1d"`, `"4h"`, `"1h"`, ... | Ranking time window |

---

## `filtrar_ativos` — Parametric Screener

Filters assets by quantitative criteria. Ideal for building dynamic watchlists.

### AI Chat Prompt

> "Filter Binance assets with a minimum price of $0.50 and minimum volume of $10 million."

### Tool Call

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

### Expected Return

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

### Screener Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `provider` | `string` | Data source (e.g., `"binance"`) |
| `min_price` | `number` | Minimum asset price |
| `max_price` | `number` | Maximum asset price |
| `min_volume` | `number` | Minimum volume in USD |
| `max_volume` | `number` | Maximum volume in USD |
| `min_change_pct` | `number` | Minimum percentage change |
| `max_change_pct` | `number` | Maximum percentage change |

---

## `salvar_filtro` — Persist a Named Screener

Repeating the same criteria every time is tedious. Save the filter and reuse it.

### AI Chat Prompt

> "Save the current filter with the name 'altcap_high_volume'."

### Tool Call

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

### Expected Return

```json
{
  "name": "altcap_high_volume",
  "salvo": true
}
```

---

## `listar_filtros` — List Saved Screeners

### AI Chat Prompt

> "List all filters I've saved."

### Tool Call

```json
{
  "tool": "listar_filtros",
  "arguments": {}
}
```

### Expected Return

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

## `excluir_filtro` — Remove a Screener

### AI Chat Prompt

> "Delete the 'pennies_com_volume' filter."

### Tool Call

```json
{
  "tool": "excluir_filtro",
  "arguments": { "name": "pennies_com_volume" }
}
```

### Expected Return

```json
{
  "excluido": true
}
```

---

## Recommended Workflow

```
top_ativos  ──>  opportunity identification
     │
     ▼
filtrar_ativos  ──>  refinement by quantitative criteria
     │
     ▼
salvar_filtro  ──>  persistence for daily reuse
     │
     ▼
listar_filtros  ──>  audit saved filters
```

1. Use `top_ativos` for a **quick market view**.
2. Use `filtrar_ativos` to **refine** by objective criteria.
3. Save useful filters with `salvar_filtro` to avoid rewriting criteria.
4. Periodically audit with `listar_filtros` and clean up with `excluir_filtro`.

---

## TypeScript SDK Example

```typescript
// Instant rankings
const topBinance = await Ct.topAtivos({
  fonte: "binance",
  janela: "1d",
});
console.log("Top gainer:", topBinance.alta[0]);

// Parametric screener
const altCoins = await Ct.filtrarAtivos({
  provider: "binance",
  minPrice: 0.50,
  minVolume: 10_000_000,
});
console.log(`${altCoins.total} assets match`);

// Persist for reuse
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

[← Back to category 03](README.en.md)
