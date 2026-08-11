# Receita 05 — Teste de Sobrevivência (Grid)

> **Nível:** Intermediário · **Premium** · **Pré-requisitos:** [Doutrina](../docs/08-doutrina/)

O teste do par: dispara compra E venda nos mesmos N momentos com o mesmo gestor.

## Passo 1 — Buscar série

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

## Passo 2 — Medir estrutura (opcional, recomendado)

```json
{ "name": "ct_medir_estrutura", "arguments": { "serie": "ct://series/binance/BTCUSDT/15m" } }
```

## Passo 3 — Teste de sobrevivência

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 20,
    "fee_pct": 0,
    "stop_r": 0.5,
    "ativacao_r": 1.0,
    "dist_r": 0.3,
    "prazo": 128,
    "breakeven": true,
    "reescala_vol": true,
    "piramide": false
  }
}
```

## Interpretação

- **`soma_pnl >= 0`:** o piso existe — a execução não sangra de lado arbitrário
- **`soma_pnl < 0`:** problema na execução ou ativo sem estrutura
- **Com fee:** se `soma_pnl < 0` com taxas, o edge gordo tem que vir do setup

## Variações

- Trocar `stop_r`, `ativacao_r`, `dist_r` para explorar a superfície
- Adicionar `piramide: true` e comparar
- Testar em outro timeframe (`1h`, `4h`)
