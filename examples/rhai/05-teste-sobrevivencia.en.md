# Recipe 05 — Survival Test (Grid)

> **Level:** Intermediate · **Premium** · **Prerequisites:** [Doctrine](../docs/08-doutrina/)

The pair test: fires buy AND sell at the same N moments with the same manager.

## Step 1 — Fetch series

```json
{ "name": "buscar_binance", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } }
```

## Step 2 — Measure structure (optional, recommended)

```json
{ "name": "ct_medir_estrutura", "arguments": { "serie": "ct://series/binance/BTCUSDT/15m" } }
```

## Step 3 — Survival test

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

## Interpretation

- **`soma_pnl >= 0`:** the floor exists — execution doesn't bleed on arbitrary side
- **`soma_pnl < 0`:** execution problem or asset without structure
- **With fee:** if `soma_pnl < 0` with fees, the fat edge must come from the setup

## Variations

- Vary `stop_r`, `ativacao_r`, `dist_r` to explore the surface
- Add `piramide: true` and compare
- Test on another timeframe (`1h`, `4h`)
