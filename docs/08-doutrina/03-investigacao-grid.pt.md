# A Investigação do Grid

> Registro da investigação do grid como fundação de gerenciamento de risco — a jornada v1→v9 refutada e o piso que ficou de pé.

## O propósito

O grid+preenchedor **não é estratégia**: é a **camada de gestão** que administra qualquer posição. O teste é de **lado arbitrário** — se a camada não sangra nem com moeda, qualquer setup com edge vira lucro por cima do piso.

## A jornada v1→v9 — todos refutados

| v | Forma | Veredito | Causa |
|---|---|---|---|
| v1 | Múltiplos fixos arbitrários | ✗ | Artefato de períodos amplos |
| v2 | Quantis de excursão | ✗ | R:R≈1, sem edge |
| v3 | EV de first-passage | ✗ | Drift do timeout, não traduz |
| v4 | Perfil + gate seletivo | ✗ | 0 trades: leitura nunca sinaliza |
| v5 | Reativo ao regime | ✗ | Instrumento×sinal conflitante |
| v6 | Assimétrico + trailing | ✗ | Declarado "passa" cedo; robustez refutou |
| v7 | v6 + pirâmide | ✗ | Toda variância de uma época |
| v8 | One-shot, 3 níveis | ✗ | Não colhe QV |
| v9 | Sessão contínua | ✗ | EV analítico ≈ −0,3r |

**Mean-reversion truncada está morta no BTC 15m** — modelo, medição e motor concordando.

## O piso que ficou — o straddle sintético

Par compra+venda com **stop curto + trailing de ativação longe** — a cauda gorda paga os stops.

- **EV +0,03 a +0,09 réguas por par**, positivo nos dois regimes de vol e em treino/holdout
- Piso de lado arbitrário: **EV ≥ 0 sem taxas — existe**. Mas é fino; fee 0,04% consome o EV
- O edge gordo tem que vir do **setup**

## Lições de método (pagas com erro)

1. **Robustez antes de declarar** (superfície + walk-forward + cross-TF + custo)
2. **Fechar o EV analítico com curvas antes de rodar o motor**
3. **Custo é de primeira classe** — fee=0 engana; turnover mata
4. **Lookahead se esconde** — Viterbi/predict_proba usam a sequência inteira
5. **Amostra enviesada engana** — os "24 maiores" contam outra história
6. **O instrumento tem que casar com o sinal**

---

> Próximo: [Fork da doutrina](./04-fork-doutrina.pt.md)
