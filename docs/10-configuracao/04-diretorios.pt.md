# Diretório de Dados

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
