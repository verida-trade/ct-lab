# Receita 08 — Microestrutura: TFI + Backtest

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Microestrutura](../docs/07-microestrutura/)

Coletar trades, calcular TFI, usar como sinal em backtest.

## Passo 1 — Coletar trades

```json
{
  "name": "coletar_trades",
  "arguments": { "provider": "binance", "symbol": "BTCUSDT", "backfill_dias": 3 }
}
```

Aguardar alguns minutos para acumular dados. Verifique status:

```json
{
  "name": "coletas_ativas",
  "arguments": {}
}
```

## Passo 2 — Calcular ct_tfi

```json
{
  "name": "ct_tfi",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/trades_1s",
    "period": 60
  }
}
```

## Passo 3 — Agregar para 15m e backtest

O trades_1s está em 1 segundo. Para usar em 15m, crie uma série OHLCV 15m e faça compose:

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

```json
{
  "name": "compor_serie",
  "arguments": {
    "name": "btc_15m_tfi",
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "columns": [
      { "source_uri": "ct://series/binance/BTCUSDT/15m", "source_column": "close", "as_column": "close" },
      { "source_uri": "ct://derived/ct_tfi_...", "source_column": "ct_tfi", "as_column": "tfi" }
    ]
  }
}
```

## Passo 4 — Backtest

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

> **Nota:** O TFI mede o desbalanceamento do fluxo de taker — valores > 0 indicam pressão compradora.
