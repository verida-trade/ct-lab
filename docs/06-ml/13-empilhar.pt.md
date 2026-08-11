# Empilhamento de Datasets

> `empilhar_datasets` — combine 2 datasets em uma base única para treino multi-série/multi-TF.

```json
{
  "componente": {},
  "entradas": ["$dataset_btc_15m", "$dataset_btc_1h"]
}
```

Os datasets são combinados ordenados por timestamp. Encadeável para N (empilhe o resultado com o próximo). Útil para treinar um modelo em múltiplos timeframes ou múltiplos ativos simultaneamente.

---

> Próximo: [Ponte backtest↔ML](./14-ponte-backtest.pt.md)
