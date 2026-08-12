# Recipe 11 — Fork the Doctrine

> **Level:** Advanced · **Premium** · **Prerequisites:** [Doctrine](../../docs/08-doutrina/)

CT Lab's default doctrine (seed CT) defines principles for building strategies. But the method is yours — you can fork the doctrine with your own rules and activate it as the default.

---

## Why fork?

| Scenario | Seed doctrine | Your doctrine |
|---|---|---|
| Min trades | 30 | You define (e.g.: 100) |
| Default fee | 0% | You define (e.g.: 0.04%) |
| Min Sharpe | — | You define (e.g.: 1.0 on holdout) |
| Preferred features | — | You list (e.g.: RSI, ATR, calendar) |
| Validation | Walk-forward | You define (e.g.: purged k-fold) |

The active doctrine is read by the agent in every session. By forking, the agent incorporates your rules automatically — no need to repeat them in chat.

---

## Step 1 — Save custom doctrine

```json
{
  "name": "salvar_doutrina",
  "arguments": {
    "tema": "ml",
    "nome": "my_ml_doctrine",
    "conteudo": "# My ML Doctrine\n\n## Principles\n1. Always use walk-forward (never simple holdout)\n2. Minimum 1000 trades to declare result\n3. Fee always 0.04% (realistic)\n4. Rule: if Sharpe < 1.0 on holdout, discard\n\n## Preferred features\n- RSI + lags\n- ATR for vol-adjusted returns\n- Calendar features (hora_sin, hora_cos)\n\n## Models\n- GBDT as baseline\n- MLP if GBDT Sharpe > 1.5\n- Never use centroide for production\n\n## Validation\n- Walk-forward with 4 folds\n- Embargo of 5 bars\n- Survival test before backtest"
  }
}
```

## Step 2 — Activate

```json
{
  "name": "aplicar_doutrina",
  "arguments": { "tema": "ml", "nome": "my_ml_doctrine" }
}
```

## Step 3 — Verify

```json
{ "uri": "ct://doutrina/ml" }
```

From now on, the agent uses YOUR doctrine instead of the CT seed in all sessions for the `ml` theme.

---

## Available themes

| Theme | What it controls |
|---|---|
| `ml` | Features, models, validation, production |
| `backtest` | Strategy, manager, metrics, fees |
| `microestrutura` | Collection, indicators, execution |
| `geral` | Cross-cutting principles |

## Variations

- **Multiple doctrines:** save several and switch with `aplicar_doutrina` as context changes
- **Versioning:** save with different names (`my_doctrine_v1`, `my_doctrine_v2`) to compare
- **Share:** export the content and send to collaborators
- **Reset:** `aplicar_doutrina` with the CT seed name reverts to default
