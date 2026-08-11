# Invocation A — Model-Guided (No UI)

> The agent reads the doctrine, decides, executes and explains — works with zero UI.

This is the "natural chat" mode — the user asks in natural language and the agent does everything:

```
User: "I want to predict BTC direction for the next 5 bars"
  → Agent reads ct://doutrina/ml (active doctrine)
  → Agent identifies: target_direcao, horizonte=5
  → Agent builds pipeline: features → dataset → split → model → evaluate
  → Agent executes via tools (montar_esteira_ml)
  → Agent explains: "Trained a GBDT with RSI+ATR+lags. 62% accuracy on holdout. Model at ct://models/btc_dir_5."
```

## What the model needs

- **Doctrine resources** (`ct://doutrina/*`) — the method
- **Agent priming** — instruction to consult doctrine and follow workflows

> Doesn't need prompts (those are user invocation) or UI. This is the baseline of operation.

---

> Next: [User-invoked invocation](./05-usuario-invoca.en.md)
