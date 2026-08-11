# Prompts e Doutrina

> Como os prompts baseline referenciam a doutrina ativa do usuário — a doutrina é a fonte única.

Os prompts **não** embutem o método — eles **referenciam** o resource `ct://doutrina/{tema}`. Isto significa:

1. O modelo lê a doutrina ativa (do usuário ou seed CT) via resource MCP
2. Os prompts baseline funcionam com qualquer doutrina ativa
3. Quando o usuário evolui a doutrina, todos os prompts automaticamente usam a nova versão

## O ciclo

```
Usuário pede "prever direção do BTC"
  → Agente lê ct://doutrina/ml (doutrina ativa do usuário)
  → Agente segue o método da doutrina
  → Agente executa: feature_set → modelo → avaliar
  → Agente explica o resultado SEGUNDO a doutrina
```

## Sem UI (Opção A — modelo-guiado)

O usuário pede em linguagem natural → o agente consulta a doutrina, decide, executa e explica. Funciona com **zero UI** — só resources + priming.

## Com UI (Opção B — usuário-invoca)

O usuário seleciona um prompt (ex.: `prever_direcao`) → form de argumentos → `prompts/get` injeta no chat. O prompt referencia a doutrina ativa automaticamente.

---

> Próximo: [Guardrails](./06-guardrails.pt.md)
