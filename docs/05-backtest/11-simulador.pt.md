# O Simulador e a Contabilidade

> Como o motor de backtest simula execução, OCO, fees e turnover.

## Modelo de execução

A cada barra, o motor:

1. Recebe a decisão da estratégia (Alvo: comprado/vendido/zerado)
2. Se a posição mudou, executa a trade no preço da barra
3. Atualiza a equity curve

### Preço de execução

- **Entrada:** close da barra onde o Alvo mudou
- **Stop/Limit (via grupo):** se o high/low da barra toca o nível, executa
- **OCO pessimista:** se num candle mais de uma saída é tocada, só a pessimista executa (modelagem conservadora sob ambiguidade OHLC)

### Fee

`fee_pct` = fração do notional por trade. Ex.: 0.001 = 0,1%.

```
fee = |qty × price| × fee_pct
```

O fee é cobrado em **cada trade** (abertura e fechamento).

---

## Turnover

Turnover = soma do notional de todos os trades. Se a estratégia opera muito (high turnover), os fees consomem o edge:

- Estratégia direcional puro: 9,3× bruto → 0,94× a 0,04% de fee
- Sempre compare `pnl_bruto` vs `pnl_total` no resultado

---

> Próximo: [Métricas](./12-metricas.pt.md)
