# Receita 11 — Fork da Doutrina

> **Nível:** Avançado · **Premium**

Crie e ative sua própria doutrina — o método é SEU.

## Passo 1 — Salvar doutrina custom

```json
{
  "name": "salvar_doutrina",
  "arguments": {
    "tema": "ml",
    "nome": "minha_doutrina_ml",
    "conteudo": "# Minha doutrina de ML\n\n## Princípios\n1. Sempre usar walk-forward (nunca holdout simples)\n2. Mínimo 1000 trades para declarar resultado\n3. Fee sempre 0.04% (realista)\n4. Regra: se Sharpe < 1.0 no holdout, descartar\n\n## Features preferidas\n- RSI + lags\n- ATR para vol-adjusted returns\n- Features de calendário\n\n## Modelos\n- GBDT como baseline\n- MLP se GBDT Sharpe > 1.5\n- Nunca usar centroide para produção"
  }
}
```

## Passo 2 — Ativar

```json
{
  "name": "aplicar_doutrina",
  "arguments": { "tema": "ml", "nome": "minha_doutrina_ml" }
}
```

## Passo 3 — Verificar

```json
// Resource: ct://doutrina/ml
// Retorna a doutrina ativa (sua versão)
```

A partir de agora, todos os prompts e o agente vão usar SUA doutrina ao invés do seed CT.
