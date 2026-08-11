# 05 · Composição de Séries

A composição de séries é o recurso que permite **combinar múltiplas séries OHLCV em uma única série derivada**, alinhando-as por timestamp (inner-join). Isto é essencial para análise cross-asset, correlação, spread trading, e alimentar modelos ML com features multivariadas.

O CT Lab oferece a ferramenta `compor_serie` para este propósito.

---

## O Problema que a Composição Resolve

Imagine que você quer analisar a relação entre BTC e ETH. Você tem:

- `ct://series/binance/BTCUSDT/15m` — 1000 candles
- `ct://series/binance/ETHUSDT/15m` — 1000 candles

Mas cada série vive isolada. Para calcular o spread `btc_close - eth_close` ou a correlação entre os retornos, você precisa de **ambas as séries alinhadas por timestamp** em uma única estrutura.

A composição faz exatamente isto: um **inner-join** por timestamp que produz uma série derivada com colunas de múltiplas fontes.

---

## `compor_serie` — Inner-Join por Timestamp

### Conceito de *Anchor*

A série derivada precisa de uma **referência temporal** — o *anchor*. O anchor define quais timestamps a série derivada conterá.

| Cenário | Regra de Anchor |
|---------|-----------------|
| 1 série raw como fonte | Auto-inferida: o anchor é a própria série raw |
| >1 série raw como fontes | Anchor **explícito** obrigatório: escolha uma das séries raw como anchor |

O anchor determina o eixo de timestamps. Todas as outras séries são alinhadas (inner-join) contra o anchor. Apenas timestamps presentes em **todas** as fontes são mantidos.

---

### Prompt de Chat IA

> "Crie uma série combinando o close do BTC e o close do ETH, alinhados por timestamp, com o BTC como anchor."

### Chamada de Ferramenta

```json
{
  "tool": "compor_serie",
  "arguments": {
    "name": "btc_eth",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      {
        "source_uri": "ct://series/binance/BTCUSDT/15m",
        "source_column": "close",
        "as_column": "btc_close"
      },
      {
        "source_uri": "ct://series/binance/ETHUSDT/15m",
        "source_column": "close",
        "as_column": "eth_close"
      }
    ]
  }
}
```

### Retorno Esperado

```json
{
  "uri": "ct://derived/btc_eth",
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

### Parâmetros

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|:-----------:|-----------|
| `name` | `string` | ✅ | Nome da série derivada (torna-se parte da URI) |
| `anchor` | `string` | ✅ | URI da série raw que serve de eixo temporal |
| `columns` | `array` | ✅ | Lista de mapeamentos coluna-origem → coluna-destino |
| `columns[].source_uri` | `string` | ✅ | URI da série de origem |
| `columns[].source_column` | `string` | ✅ | Coluna na série de origem (`open`, `high`, `low`, `close`, `volume`) |
| `columns[].as_column` | `string` | ✅ | Nome da coluna na série derivada |

### Retorno

| Campo | Descrição |
|-------|-----------|
| `uri` | URI da série derivada: `ct://derived/<name>` |
| `row_count` | Número de linhas após o inner-join |
| `first_ts` / `last_ts` | Extremidades temporais |
| `anchor_uri` | URI da série usada como anchor |

> **Nota:** `row_count` pode ser **menor** que o número de candles de qualquer série individual, pois o inner-join mantém apenas timestamps presentes em **todas** as fontes.

---

## Exemplo Completo: Spread BTC/ETH

Vamos construir um exemplo completo end-to-end: ingerir BTC e ETH, compô-los, e verificar o resultado.

### Passo 1: Ingerir BTCUSDT

```json
{
  "tool": "buscar_binance",
  "arguments": { "symbol": "BTCUSDT", "interval": "15m" }
}
```

Retorno:
```json
{ "uri": "ct://series/binance/BTCUSDT/15m", "row_count": 1000 }
```

### Passo 2: Ingerir ETHUSDT

```json
{
  "tool": "buscar_binance",
  "arguments": { "symbol": "ETHUSDT", "interval": "15m" }
}
```

Retorno:
```json
{ "uri": "ct://series/binance/ETHUSDT/15m", "row_count": 1000 }
}
```

### Passo 3: Compor a Série

```json
{
  "tool": "compor_serie",
  "arguments": {
    "name": "btc_eth_spread",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      {
        "source_uri": "ct://series/binance/BTCUSDT/15m",
        "source_column": "close",
        "as_column": "btc_close"
      },
      {
        "source_uri": "ct://series/binance/ETHUSDT/15m",
        "source_column": "close",
        "as_column": "eth_close"
      }
    ]
  }
}
```

Retorno:
```json
{
  "uri": "ct://derived/btc_eth_spread",
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

> 950 linhas — as 50 diferenças são timestamps onde ETH não tinha candle correspondente.

### Passo 4: Ler os Dados via Resource

A série derivada pode ser lida como qualquer outra série:

```
ct://derived/btc_eth_spread/tail/5
```

Resposta (resource):
```json
{
  "uri": "ct://derived/btc_eth_spread/tail/5",
  "row_count": 5,
  "columns": ["btc_close", "eth_close"],
  "timestamps": [
    "2026-08-11T14:00:00Z",
    "2026-08-11T14:15:00Z",
    "2026-08-11T14:30:00Z",
    "2026-08-11T14:45:00Z",
    "2026-08-11T15:00:00Z"
  ],
  "rows": [
    { "btc_close": 64000.0, "eth_close": 3200.0 },
    { "btc_close": 64100.0, "eth_close": 3210.0 },
    { "btc_close": 63950.0, "eth_close": 3185.0 },
    { "btc_close": 64050.0, "eth_close": 3195.0 },
    { "btc_close": 64080.0, "eth_close": 3205.0 }
  ]
}
```

### Passo 5: Verificar com `info_serie`

```json
{
  "tool": "info_serie",
  "arguments": { "uri": "ct://derived/btc_eth_spread" }
}
```

Retorno:
```json
{
  "uri": "ct://derived/btc_eth_spread",
  "kind": "derived",
  "columns": ["btc_close", "eth_close"],
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "last_accessed_at": "2026-08-11T15:12:00Z",
  "source_uris": [
    "ct://series/binance/BTCUSDT/15m",
    "ct://series/binance/ETHUSDT/15m"
  ]
}
```

> Note que `kind` é `"derived"` e `source_uris` lista as séries originais.

---

## Adicionando Mais Colunas

Você pode puxar múltiplas colunas da mesma série:

```json
{
  "tool": "compor_serie",
  "arguments": {
    "name": "btc_eth_full",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close",  "as_column": "btc_close" },
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "volume", "as_column": "btc_volume" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close",  "as_column": "eth_close" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "volume", "as_column": "eth_volume" }
    ]
  }
}
```

Retorno:
```json
{
  "uri": "ct://derived/btc_eth_full",
  "row_count": 950,
  "first_ts": "2026-08-10T18:00:00Z",
  "last_ts": "2026-08-11T15:00:00Z",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

---

## Exemplo em TypeScript

```typescript
// Ingerir ambas as séries
await Ct.buscarBinance({ symbol: "BTCUSDT", interval: "15m" });
await Ct.buscarBinance({ symbol: "ETHUSDT", interval: "15m" });

// Compor
const composed = await Ct.comporSerie({
  name: "btc_eth_spread",
  anchor: "ct://series/binance/BTCUSDT/15m",
  columns: [
    {
      sourceUri: "ct://series/binance/BTCUSDT/15m",
      sourceColumn: "close",
      asColumn: "btc_close",
    },
    {
      sourceUri: "ct://series/binance/ETHUSDT/15m",
      sourceColumn: "close",
      asColumn: "eth_close",
    },
  ],
});
console.log(`Série derivada: ${composed.uri}`);
console.log(`${composed.row_count} linhas alinhadas`);
```

---

## Casos de Uso

| Caso de Uso | Como a Composição Ajuda |
|-------------|------------------------|
| Spread trading (BTC-ETH) | Alinha closes de dois ativos em uma série |
| Correlação multi-asset | Combina N closes para matriz de correlação |
| Features para ML | Junta features de múltiplas séries em uma derive |
| Pair trading | Alinha preços para calcular ratio e desvio |

---

[← Voltar para a categoria 03](README.pt.md)
