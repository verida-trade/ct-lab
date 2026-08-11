# Data Directory

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
