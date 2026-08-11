# O Teste de Sobrevivência

> A régua do método CT: se a execução não sangra nem com moeda, qualquer setup com edge vira lucro.

## O instrumento

O "par" = duas execuções independentes (compra e venda) sobre os mesmos N momentos (20 canônico). O termo direcional cancela por construção — o que sobra mede a **execução**.

## Veredito

**Σ líquido (sem taxas) ≥ 0** no agregado = o piso existe.

Com taxas é o teste do mundo real do setup.

## Como executar

Veja [Teste de Sobrevivência (Grid) na seção Backtest](../05-backtest/08-teste-sobrevivencia.pt.md) para a referência completa da tool `ct_testar_sobrevivencia`.

## Interpretação

| Resultado | Diagnóstico |
|---|---|
| Σ ≥ 0 (sem taxas) | O piso existe — a execução não sangra |
| Σ < 0 (sem taxas) | Execução/gestão ruim ou ativo sem estrutura |
| Σ ≥ 0 sem taxas, < 0 com taxas | Turnover mata — o edge gordo tem que vir do setup |
| Setup não supera o piso | A leitura não trouxe edge — fica o baseline |

---

> Próximo: [A investigação do grid](./03-investigacao-grid.pt.md)
