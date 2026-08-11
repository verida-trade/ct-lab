# Estratégia Declarativa (Inline)

> A forma mais simples de backtest: um script Rhai inline passado em `estrategia_script`.

Não é preciso criar arquivos externos — a estratégia inteira cabe em uma string.

---

## Exemplo 1: Cruzamento de médias

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"sma9\"][0] > ind[\"sma21\"][0] { comprado(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "sma9": { "receita": "sma(close, 9)" },
      "sma21": { "receita": "sma(close, 21)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "sma_cross"
  }
}
```

## Exemplo 2: RSI oversold/overbought

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < 30.0 { comprado(1.0) } else if ind[\"rsi\"][0] > 70.0 { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, 14)" }
    },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_mean_reversion"
  }
}
```

## Exemplo 3: Com parâmetros

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia_script": "if ind[\"rsi\"][0] < par[\"oversold\"] { comprado(1.0) } else if ind[\"rsi\"][0] > par[\"overbought\"] { vendido(1.0) } else { zerado() }",
    "indicadores_receitas": {
      "rsi": { "receita": "rsi(close, par[\"p\"])" }
    },
    "parametros": { "p": 14, "oversold": 25.0, "overbought": 75.0 },
    "capital_inicial": 1000,
    "fee_pct": 0.001,
    "nome": "rsi_tight"
  }
}
```

## Estratégia via arquivo (URI)

Em vez de inline, referencie um arquivo:

```json
{
  "name": "ct_backtest",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "estrategia": "file:///path/to/minha_estrategia.rhai",
    "estrategia_hash": "sha256hex...",
    "capital_inicial": 1000
  }
}
```

- `estrategia_hash` é opcional para fixar a integridade do script.

---

> Próximo: [Estratégia em Rhai (avançada)](./03-estrategia-rhai.pt.md)
