# Arquivo `.cthints`

O `.cthints` é um arquivo de hints por projeto que o agente lê para entender contexto:

```json
{
  "project": "meu-projeto-ct",
  "description": "Estratégias de mean-reversion em BTC 15m",
  "timeframe": "15m",
  "symbols": ["BTCUSDT", "ETHUSDT"],
  "risk_tolerance": "conservative"
}
```

Coloque `.cthints` na raiz do seu projeto. O agente incorpora o contexto automaticamente.
