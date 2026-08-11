# Transformação Custom (Python)

> `transformar_custom` — pré-processamento fit-dependente via script Python.

Diferente de `transformar_coluna` (stateless), `transformar_custom` aprende parâmetros no treino e os reaplica no serving.

## Contrato

```python
def ajustar(X, hp):
    # Aprende parâmetros do treino (ex: mediana para imputação custom)
    # Retorna: (modelo_ajuste, X_transformado)
    ...

def transformar(modelo_ajuste, X):
    # Reaplica o ajuste learned em novos dados
    # Retorna: X_transformado
    ...
```

## Uso

```json
{
  "componente": {
    "script": "import numpy as np\n\ndef ajustar(X, hp):\n    mediana = np.median(X, axis=0)\n    X_t = X - mediana\n    return {'mediana': mediana.tolist()}, X_t\n\ndef transformar(ajuste, X):\n    return X - np.array(ajuste['mediana'])",
    "deps": ["numpy>=2.0"],
    "hiperparametros": {}
  }
}
```

---

> Voltar para: [README](./README.pt.md)
