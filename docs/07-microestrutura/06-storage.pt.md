# Volume e Storage

> Planning: quanto disco a microestrutura consome.

## Por símbolo/ano (raw)

| Conjunto | Por símbolo/ano | 100 símbolos/ano | Parquet+ZSTD (~5×) |
|---|---|---|---|
| `trades_1s` (41 cols) | ~10 GB | ~1 TB | ~200 GB |
| `book_1s` (51 cols) | ~13 GB | ~1.3 TB | ~260 GB |
| **Total** | **~23 GB** | **~2.3 TB** | **~460 GB** |

## Exemplos práticos

| Configuração | Disco/ano (comprimido) |
|---|---|
| 10 símbolos | ~45 GB |
| 30 símbolos | ~140 GB |

## Diretório

`CT_MCP_STREAMS_DIR` > `<exe_dir>/streams/` > tempdir fallback.

Parquet particionado: `{streams_dir}/trades_1s/{provider}/{symbol}/{YYYY-MM-DD}.parquet`

---

> Próximo: [Gateway WebSocket](./07-gateway.pt.md)
