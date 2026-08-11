# FAQ — ML Errors

## `uv` not found

```
ERROR: CT_MCP_UV=uv not found in PATH
```

**Solution:** Install `uv`:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## Python version mismatch

```
ERROR: Python 3.14.5 not available
```

Override via env: `CT_MCP_ML_PYTHON=3.12.0`

## Deps not resolved

Check version conflicts. Use `deps: ["scikit-learn==1.9.0"]` to pin compatible versions.

## Model doesn't generalize (overfitting)

Symptoms: high training accuracy, low holdout accuracy.

**Solutions:**
- Reduce GBDT `max_depth`
- Increase regularization
- Use walk-forward instead of simple holdout
- Add embargo to split (`embargo: 10`)
