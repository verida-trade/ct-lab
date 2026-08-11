# Indicadores Custom (Rhai e Python)

> Crie indicadores totalmente personalizados usando scripts Rhai ou Python dentro da pipeline.

O step `custom` na pipeline permite executar lógica arbitrária — seja em Rhai (linguagem embutida, sem dependências) ou Python (via `uv`, com bibliotecas).

---

## Rhai custom (inline)

Mais rápido — sem ambiente externo, executa direto no processo do servidor.

```json
{
  "id": "meu_sinal",
  "operacao": "custom",
  "script": "let r = rsi(close, par[\"p\"]); let m = sma(close, par[\"m\"]); if r > 70.0 && close[0] > m[0] { -1.0 } else if r < 30.0 && close[0] < m[0] { 1.0 } else { 0.0 }",
  "entradas": [
    { "alias": "close", "fonte": "$anchor", "coluna": "close" },
    { "alias": "high", "fonte": "$anchor", "coluna": "high" },
    { "alias": "low", "fonte": "$anchor", "coluna": "low" }
  ],
  "parametros": { "p": 14, "m": 20 },
  "coluna_saida": "sinal"
}
```

### Como funciona

- `entradas` mapeia alias → série/coluna. No script, acesse via `ent["alias"]` ou `close`, `high`, etc. (se o alias for nome de coluna nativa).
- `parametros` expõe valores como `par["nome"]` no script.
- `script` é uma expressão Rhai. Para retornar múltiplas colunas, use um mapa: `#{ "col1": ..., "col2": ... }`.

### Rhai via URI

Em vez de inline, referencie um arquivo `.rhai`:

```json
{
  "id": "custom",
  "operacao": "custom",
  "uri": "file:///path/to/meu_indicador.rhai",
  "entradas": [...],
  "parametros": {...}
}
```

---

## Python custom (inline)

Para lógica que precisa de bibliotecas Python (numpy, pandas, scipy, etc.):

```json
{
  "id": "zscore_py",
  "operacao": "custom",
  "script": "import numpy as np\n\ndef calcular(close, high, low, volume, par):\n    r = np.array(close)\n    m = np.mean(r)\n    s = np.std(r)\n    z = (r - m) / s if s > 0 else np.zeros_like(r)\n    return {\"zscore\": z.tolist()}",
  "entradas": [
    { "alias": "close", "fonte": "$anchor", "coluna": "close" }
  ],
  "parametros": {}
}
```

### Ambiente Python

O `ct-mcp-server` executa Python via `uv` em um ambiente efêmero controlado:

- **Versão fixa:** `CT_MCP_ML_PYTHON` (default `3.14.5`)
- **Deps:** bibliotecas base por família (`gbdt` → scikit-learn, `mlp` → torch). Para scripts custom simples, `numpy` vem no ambiente base.
- **Deps extras:** passe `deps: ["scipy==1.14.0"]` no step para adicionar bibliotecas.
- **Sem fallback:** se `uv` não estiver instalado, o servidor retorna erro com instruções de instalação — não degrada para o Python do host.

```json
{
  "id": "custom_scipy",
  "operacao": "custom",
  "script": "from scipy.signal import find_peaks\n\ndef calcular(close, par):\n    peaks, _ = find_peaks(close, distance=par[\"dist\"])\n    result = [0.0] * len(close)\n    for p in peaks:\n        result[p] = 1.0\n    return {\"peaks\": result}",
  "deps": ["scipy==1.14.0"],
  "entradas": [{ "alias": "close", "fonte": "$anchor", "coluna": "close" }],
  "parametros": { "dist": 5 }
}
```

### Python via URI

```json
{
  "id": "custom",
  "operacao": "custom",
  "uri": "file:///path/to/meu_indicador.py",
  "deps": ["pandas==2.2"],
  "entradas": [...]
}
```

---

## Contrato do script Python

O script Python deve definir uma função `calcular` que recebe as entradas como arrays Python (lists) e retorna um dict `{ "coluna": [valores] }`:

```python
def calcular(close, high, low, volume, par):
    # par é um dict com os parâmetros
    # close, high, low, volume são lists de floats
    result = [c * par.get("factor", 1.0) for c in close]
    return {"custom_col": result}
```

---

## Quando usar Rhai vs Python

| Critério | Rhai | Python |
|---|---|---|
| Velocidade | ⚡ Mais rápido (in-process) | ⏳ Overhead de startup do `uv` |
| Dependências | Nenhuma (embutido) | Qualquer lib Python via `uv` |
| Complexidade | Expressões simples/médias | Cálculos complexos, matrizes, ML |
| Estado | Stateless (por expressão) | Stateless (por call) |
| Melhor para | Sinais, filtros, álgebra | Estatística, ML, transformações complexas |

---

> Próximo: [Pipeline declarativo](./03-pipeline-declarativo.pt.md) · [Rhai vetorizado](./05-rhai-vetorizado.pt.md)
