# Dataset Stacking

> `empilhar_datasets` — combine 2 datasets into a single base for multi-series/multi-TF training.

```json
{
  "componente": {},
  "entradas": ["$dataset_btc_15m", "$dataset_btc_1h"]
}
```

Datasets are combined ordered by timestamp. Chainable for N (stack the result with the next). Useful for training a model on multiple timeframes or multiple assets simultaneously.

---

> Next: [Backtest↔ML bridge](./14-ponte-backtest.en.md)
