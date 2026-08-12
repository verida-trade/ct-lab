# Variáveis de Ambiente

> Referência completa de env vars e diretórios de dados do CT Lab e ct-mcp-server.

## Diretórios de dados

O CT Lab armazena dados em:

| Plataforma | Caminho |
|---|---|
| macOS | `~/Library/Application Support/CT/ct-lab/` |
| Linux | `~/.config/ct-lab/` |
| Windows | `%APPDATA%\CT\ct-lab\` |

Override via env: `CT_PATH_ROOT=/path/to/custom`

O `ct-mcp-server` armazena séries em:
- SQLite: `<exe_dir>/ct_mcp.db` (OHLCV)
- Parquet: `<exe_dir>/streams/` (microestrutura)
- Override: `CT_MCP_STREAMS_DIR=/path/to/streams`

## CT Lab

| Env | Default | Descrição |
|---|---|---|
| `CT_PROVIDER` | — | Provider de IA: `openai`, `anthropic`, `google`, `ollama` |
| `CT_MODEL` | — | Modelo do provider |
| `CT_MODE` | `auto` | `auto` (agente) ou `chat` (conversa) |
| `CT_PATH_ROOT` | — | Diretório de dados (override em testes) |

## ct-mcp-server

| Env | Default | Descrição |
|---|---|---|
| `CT_MCP_UV` | `uv` | Caminho do binário `uv` |
| `CT_MCP_ML_PYTHON` | `3.14.5` | Versão do Python para ML |
| `CT_MCP_STREAMS_DIR` | `<exe>/streams/` | Diretório de Parquet de streams |
| `CT_MCP_DIAG` | — | Se setada, inclui primitivas diag |
| `CT_MCP_BINANCE_BASE` | — | Override da base URL da Binance Spot |
| `CT_MCP_BINANCE_UM_BASE` | — | Override da base URL da Binance Futures |
| `CT_MCP_BINANCE_WS_BASE` | — | Override da WS base URL |
| `CT_MCP_BINANCE_BULK_BASE` | — | Override da URL de bulk dumps |
| `CT_MCP_BINANCE_TIMEOUT_MS` | — | Timeout HTTP da Binance |
| `CT_MCP_FETCH_RETRY_BASE_MS` | `1000` | Base do backoff de retry HTTP |

---

> Voltar para: [README](./README.pt.md)
