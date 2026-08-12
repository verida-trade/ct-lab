# Receita 11 — Fork da Doutrina

> **Nível:** Avançado · **Premium** · **Pré-requisitos:** [Doutrina](../../docs/08-doutrina/)

A doutrina padrão do CT Lab (seed CT) define princípios de construção de estratégias. Mas o método é seu — você pode forkar a doutrina com suas próprias regras e ativar como padrão.

---

## Por que forkar?

| Cenário | Doutrina seed | Sua doutrina |
|---|---|---|
| Mínimo de trades | 30 | Você define (ex: 100) |
| Fee padrão | 0% | Você define (ex: 0,04%) |
| Sharpe mínimo | — | Você define (ex: 1,0 no holdout) |
| Features preferidas | — | Você lista (ex: RSI, ATR, calendário) |
| Validação | Walk-forward | Você define (ex: purged k-fold) |

A doutrina ativa é lida pelo agente em toda sessão. Ao forkear, o agente incorpora suas regras automaticamente — sem precisar repeti-las no chat.

---

## Passo 1 — Salvar doutrina custom

```json
{
  "name": "salvar_doutrina",
  "arguments": {
    "tema": "ml",
    "nome": "minha_doutrina_ml",
    "conteudo": "# Minha doutrina de ML\n\n## Princípios\n1. Sempre usar walk-forward (nunca holdout simples)\n2. Mínimo 1000 trades para declarar resultado\n3. Fee sempre 0,04% (realista)\n4. Regra: se Sharpe < 1,0 no holdout, descartar\n\n## Features preferidas\n- RSI + lags\n- ATR para vol-adjusted returns\n- Features de calendário (hora_sin, hora_cos)\n\n## Modelos\n- GBDT como baseline\n- MLP se GBDT Sharpe > 1,5\n- Nunca usar centroide para produção\n\n## Validação\n- Walk-forward com 4 folds\n- Embargo de 5 barras\n- Survival test antes de backtest"
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
{ "uri": "ct://doutrina/ml" }
```

A partir de agora, o agente usa SUA doutrina ao invés do seed CT em todas as sessões do tema `ml`.

---

## Temas disponíveis

| Tema | O que controla |
|---|---|
| `ml` | Features, modelos, validação, produção |
| `backtest` | Estratégia, gestor, métricas, fees |
| `microestrutura` | Coleta, indicadores, execução |
| `geral` | Princípios cross-cutting |

## Variações

- **Múltiplas doutrinas:** salve várias e troque com `aplicar_doutrina` conforme o contexto
- **Versionamento:** salve com nomes diferentes (`minha_doutrina_v1`, `minha_doutrina_v2`) para comparar
- **Compartilhar:** exporte o conteúdo e envie para colaboradores
- **Reset:** `aplicar_doutrina` com nome do seed CT volta ao padrão
