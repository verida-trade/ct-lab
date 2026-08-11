# Recipe 11 — Fork the Doctrine

> **Level:** Advanced · **Premium**

Create and activate your own doctrine — the method is YOURS.

## Step 1 — Save custom doctrine

```json
{
  "name": "salvar_doutrina",
  "arguments": {
    "tema": "ml",
    "nome": "my_ml_doctrine",
    "conteudo": "# My ML doctrine\n\n## Principles\n1. Always use walk-forward (never simple holdout)\n2. Minimum 1000 trades to declare result\n3. Fee always 0.04% (realistic)\n4. Rule: if Sharpe < 1.0 on holdout, discard\n\n## Preferred features\n- RSI + lags\n- ATR for vol-adjusted returns\n- Calendar features"
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
// Resource: ct://doutrina/ml
// Returns the active doctrine (your version)
```

From now on, all prompts and the agent will use YOUR doctrine instead of the CT seed.
