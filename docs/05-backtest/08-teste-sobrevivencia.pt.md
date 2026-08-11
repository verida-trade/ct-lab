# Teste de Sobrevivência (Grid)

> **Premium.** O teste do par: dispara compra E venda nos mesmos N momentos com o mesmo gestor. Se a camada não sangra nem com moeda, qualquer setup com edge vira lucro por cima do piso.

A fundação do método CT: **sobrevivência de lado arbitrário**. A execução não pode depender de acertar lado.

---

## O que é

O "par" do teste = **duas execuções independentes** (compra e venda) sobre os mesmos N momentos. O termo direcional cancela por construção — o que sobra mede a **execução**, não o palpite.

```
Momento 1:  Compra → gestor → P&L_1c    |    Venda → gestor → P&L_1v
Momento 2:  Compra → gestor → P&L_2c    |    Venda → gestor → P&L_2v
...
Momento N:  ...                          |    ...
Veredito:   Σ(P&L_c) + Σ(P&L_v) ≥ 0  (sem taxas)
```

---

## Chamada

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 20,
    "fee_pct": 0,
    "stop_r": 0.5,
    "ativacao_r": 1.0,
    "dist_r": 0.3,
    "prazo": 128,
    "breakeven": true,
    "reescala_vol": true,
    "piramide": false
  }
}
```

### Parâmetros

| Parâmetro | Default | Descrição |
|---|---|---|
| `serie` | obrigatório | URI da série OHLCV |
| `momentos` | 20 | N de momentos espaçados no período |
| `fee_pct` | 0 | Fee por trade (0 = sem taxas) |
| `stop_r` | 0.5 | Stop em réguas (amplitude local) |
| `ativacao_r` | 1.0 | Ativação do trailing em réguas |
| `dist_r` | 0.3 | Distância do trailing em réguas |
| `prazo` | 128 | Fecha a mercado após N barras |
| `breakeven` | false | Move stop para entry quando favorável |
| `reescala_vol` | false | Reescala com volatilidade local |
| `piramide` | false | Piramida fora de vol alta |

---

## Retorno

```json
{
  "soma_pnl": 0.034,
  "ev_par_reguas": 0.063,
  "pares_positivos": 12,
  "por_momento": [
    { "momento": 1, "compra_pnl": 0.1, "venda_pnl": -0.05 },
    { "momento": 2, "compra_pnl": -0.03, "venda_pnl": 0.08 },
    ...
  ]
}
```

| Campo | Significado |
|---|---|
| `soma_pnl` | Σ líquido (sem taxas) — o veredito |
| `ev_par_reguas` | EV por par em réguas |
| `pares_positivos` | Quantos dos N momentos tiveram soma ≥ 0 |
| `por_momento` | P&L de cada execução (compra e venda) por momento |

---

## Veredito

- **Σ ≥ 0 (sem taxas):** o piso existe. A camada de execução não sangra.
- **Σ < 0:** o problema é execução/gestão (grupo/gestor) ou o ativo não tem estrutura.
- **Com taxas:** se Σ < 0 com fee, o edge gordo tem que vir do setup.

---

## Diagnóstico por camada na reprovação

| Sintoma | Diagnóstico | Ação |
|---|---|---|
| Piso não passa | Execução/gestão ruim ou ativo sem estrutura | Confira `ct_medir_estrutura`; ajuste stop/trailing |
| Piso OK mas setup não supera | Leitura não trouxe edge | Troque ou ajuste a leitura |
| Piso OK com taxas, sem taxas não | Turnover mata | Reduza frequência |

---

## Na UI do CT Lab

A aba **Grid** da tela de backtest faz o mesmo teste com dois cliques: escolha o período por data, marque as flags das regras adaptativas, veja o veredito e a tabela por momento.

---

> Próximo: [Comparação de backtests](./09-comparacao.pt.md) · [Doutrina CT](../08-doutrina/)
