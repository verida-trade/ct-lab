# Compose Cross-Asset

> Junte séries de ativos diferentes por timestamp. Essencial para análise cross-asset, spreads e correlações.

O `compor_serie` faz um **inner-join** de N séries raw por timestamp. O resultado é uma série **synthetic** com colunas de múltiplos ativos alinhadas.

---

## Quando usar

| Situação | Exemplo |
|---|---|
| Spread entre ativos | BTC close − ETH close |
| Razão entre ativos | BTC / ETH |
| Correlação rolling | Correlação de 30 bars entre BTC e ouro |
| Análise de par | EUR/USD vs GBP/USD |
| Multi-asset features para ML | BTC + ETH + SOL features no mesmo dataset |

---

## Chamada básica

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "btc_eth_15m",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc_close" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth_close" }
    ]
  }
}
```

**Retorno:**
```json
{
  "uri": "ct://derived/btc_eth_15m",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "anchor_uri": "ct://series/binance/BTCUSDT/15m"
}
```

---

## O parâmetro `anchor`

O anchor define a **linha do tempo** do output:

- **1 raw entre as fontes:** anchor é auto-inferido
- **>1 raw entre as fontes:** anchor é obrigatório

Sempre que você compor séries de ativos diferentes, haverá >1 raw — anchor é obrigatório. Escolha a série com mais dados (ou a mais relevante) como anchor.

---

## Exemplo: Spread BTC-ETH

### Passo 1 — Compor

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "spread_btc_eth",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth" }
    ]
  }
}
```

### Passo 2 — Calcular o spread via Rhai

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://derived/spread_btc_eth",
    "name": "spread_value",
    "receita": "btc - eth"
  }
}
```

### Passo 3 — Calcular a razão

```json
{
  "name": "materializar_indicador",
  "arguments": {
    "fonte": "ct://derived/spread_btc_eth",
    "name": "ratio_btc_eth",
    "receita": "btc / eth"
  }
}
```

---

## Exemplo: 3 ativos alinhados

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "tri_crypto",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "btc" },
      { "source_uri": "ct://series/binance/ETHUSDT/15m", "source_column": "close", "as_column": "eth" },
      { "source_uri": "ct://series/binance/SOLUSDT/15m", "source_column": "close", "as_column": "sol" }
    ]
  }
}
```

Depois calcule features sobre a série synthetic:

```rhai
// Correlação BTC-ETH (aproximada via média de produto dos desvios)
let btc_r = (btc - avg(btc));
let eth_r = (eth - avg(eth));
btc_r * eth_r
```

---

## Compose dentro de pipeline

Você também pode fazer compose dentro da pipeline usando o op `compose`:

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "multi_asset_signal",
  "output": "$sinal",
  "steps": [
    { "id": "btc", "op": "sma", "source": "$anchor", "period": 20 },
    { "id": "eth", "op": "sma", "source": "ct://series/binance/ETHUSDT/15m", "period": 20 },
    {
      "id": "merge",
      "op": "compose",
      "columns": [
        { "source": "$btc", "source_column": "sma", "as_column": "btc_sma" },
        { "source": "$eth", "source_column": "sma", "as_column": "eth_sma" }
      ]
    },
    { "id": "sinal", "op": "custom", "script": "if ent[\"btc_sma\"] > ent[\"eth_sma\"] { 1.0 } else { -1.0 }", "entradas": [{"alias":"btc_sma","fonte":"$merge","coluna":"btc_sma"},{"alias":"eth_sma","fonte":"$merge","coluna":"eth_sma"}], "coluna_saida": "sinal" }
  ]
}
```

> **Nota:** para referenciar uma URI externa no `source` de um step, use a URI completa em vez de `$anchor`.

---

> Próximo: [Indicadores custom](./09-indicadores-custom.pt.md) · [Pipeline declarativo](./03-pipeline-declarativo.pt.md)
