# Invocação A — Modelo-Guiado (Sem UI)

> O agente lê a doutrina, decide, executa e explica — funciona com zero UI.

Este é o modo "chat natural" — o usuário pede em linguagem natural e o agente faz tudo:

```
Usuário: "Quero prever a direção do BTC nos próximos 5 bars"
  → Agente lê ct://doutrina/ml (doutrina ativa)
  → Agente identifica: target_direcao, horizonte=5
  → Agente monta a esteira: features → dataset → split → modelo → avaliar
  → Agente executa via tools (montar_esteira_ml)
  → Agente explica: "Treinei um GBDT com RSI+ATR+lags. Acurácia 62% no holdout. O modelo está em ct://models/btc_dir_5."
```

## O que o modelo precisa

- **Resources de doutrina** (`ct://doutrina/*`) — o método
- **Priming do agente** — instrução para consultar a doutrina e seguir os workflows

> Não precisa de prompts (são invocação do usuário) nem UI. É o piso de funcionamento.

---

> Próximo: [Invocação usuário-invoca](./05-usuario-invoca.pt.md)
