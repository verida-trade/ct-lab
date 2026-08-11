# Prompts & Doctrine

> How baseline prompts reference the user's active doctrine — the doctrine is the single source.

Prompts **don't** embed the method — they **reference** the `ct://doutrina/{tema}` resource. This means:

1. The model reads the active doctrine (user's or CT seed) via MCP resource
2. Baseline prompts work with any active doctrine
3. When the user evolves the doctrine, all prompts automatically use the new version

## The cycle

```
User asks "predict BTC direction"
  → Agent reads ct://doutrina/ml (user's active doctrine)
  → Agent follows the doctrine's method
  → Agent executes: feature_set → model → evaluate
  → Agent explains the result ACCORDING to the doctrine
```

## Without UI (Option A — model-guided)

The user asks in natural language → the agent consults the doctrine, decides, executes and explains. Works with **zero UI** — just resources + priming.

## With UI (Option B — user-invoked)

The user selects a prompt (e.g., `prever_direcao`) → argument form → `prompts/get` injects into chat. The prompt references the active doctrine automatically.

---

> Next: [Guardrails](./06-guardrails.en.md)
