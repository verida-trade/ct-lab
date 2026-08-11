# Guardrails — Anti-Auto-Engano

> Lições pagas com erro na investigação do grid. Checklist de robustez antes de declarar qualquer resultado.

## Checklist de robustez

Antes de declarar que uma estratégia "funciona", verifique:

- [ ] **Holdout intocado:** treino e validação separados temporalmente, sem leakage
- [ ] **Walk-forward:** múltiplas janelas de validação, não apenas uma
- [ ] **Superfície de parâmetros:** o resultado não é um pico isolado — há uma região contígua e monotônica
- [ ] **Cross-TF:** o resultado generaliza para outro timeframe
- [ ] **Custo realista:** fee ≠ 0 (0,04% típico); turnover mata frequentemente
- [ ] **Out-of-sample:** leitura de regime usada como setup foi testada fora da amostra de treino

## Armadilhas conhecidas

| Armadilha | Como aparece | Como evitar |
|---|---|---|
| **Lookahead** | Viterbi/predict_proba usam a sequência inteira | Usar versão causal (predict last) |
| **Drift do timeout** | "Fechar a mercado no timeout do teste" credita drift | Fechar a mercado no preço real da barra |
| **Amostra enviesada** | Os "24 maiores episódios" contam outra história | Testar no agregado, não selecionado |
| **Métrica composta esconde** | Lag 0 do ct_regime era picote | Olhar métricas individuais |
| **Estratégia carrega estado** | `EstrategiaRhai` carrega estado entre backtests | Recompilar por backtest |
| **Fee=0 engana** | Direcional puro: 9,3× bruto → 0,94× a 0,04% | Sempre testar com fee realista |

---

> Voltar para: [README](./README.pt.md)
