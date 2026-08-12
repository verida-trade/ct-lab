# Environment Variables

> Complete reference of env vars and data directories for CT Lab and ct-mcp-server.

## Data directories

CT Lab stores data at:

| Platform | Path |
|---|---|
| macOS | `~/Library/Application Support/CT/ct-lab/` |
| Linux | `~/.config/ct-lab/` |
| Windows | `%APPDATA%\CT\ct-lab\` |

Override via env: `CT_PATH_ROOT=/path/to/custom`

The `ct-mcp-server` stores series at:
- SQLite: `<exe_dir>/ct_mcp.db` (OHLCV)
- Parquet: `<exe_dir>/streams/` (microstructure)
- Override: `CT_MCP_STREAMS_DIR=/path/to/streams`

## CT Lab

| Env | Default | Description |
|---|---|---|
| `CT_PROVIDER` | — | AI provider: `openai`, `anthropic`, `google`, `ollama` |
| `CT_MODEL` | — | Provider model |
| `CT_MODE` | `auto` | `auto` (agent) or `chat` (conversation) |
| `CT_PATH_ROOT` | — | Data directory (test override) |

## ct-mcp-server

| Env | Default | Description |
|---|---|---|
| `CT_MCP_UV` | `uv` | Path to `uv` binary |
| `CT_MCP_ML_PYTHON` | `3.14.5` | Python version for ML |
| `CT_MCP_STREAMS_DIR` | `<exe>/streams/` | Parquet streams directory |
| `CT_MCP_DIAG` | — | If set, includes diag primitives |
| `CT_MCP_BINANCE_BASE` | — | Binance Spot base URL override |
| `CT_MCP_BINANCE_UM_BASE` | — | Binance Futures base URL override |
| `CT_MCP_BINANCE_WS_BASE` | — | WS base URL override |
| `CT_MCP_BINANCE_BULK_BASE` | — | Bulk dumps URL override |
| `CT_MCP_BINANCE_TIMEOUT_MS` | — | Binance HTTP timeout |
| `CT_MCP_FETCH_RETRY_BASE_MS` | `1000` | HTTP retry backoff base |

---

> Back to: [README](./README.en.md)
