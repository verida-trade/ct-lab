# FAQ — Erros de ML

## `uv` não encontrado

```
ERROR: CT_MCP_UV=uv not found in PATH
```

**Solução:** Instale o `uv`:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## Versão do Python incompatível

```
ERROR: Python 3.14.5 not available
```

O servidor pinou Python 3.14.5. Override via env: `CT_MCP_ML_PYTHON=3.12.0`

## Deps não resolvidas

```
ERROR: uv failed to resolve dependencies
```

Verifique conflitos de versão. Use `deps: ["scikit-learn==1.9.0"]` para fixar versões compatíveis.

## Modelo não generaliza (overfitting)

Sintomas: alta acurácia no treino, baixa no holdout.

**Soluções:**
- Reduza `max_depth` do GBDT
- Aumente regularization
- Use walk-forward em vez de holdout simples
- Adicione embargo no split (`embargo: 10`)
