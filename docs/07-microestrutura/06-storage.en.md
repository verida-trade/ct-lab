# Volume & Storage

> Planning: how much disk microstructure consumes.

## Per symbol/year (raw)

| Set | Per symbol/year | 100 symbols/year | Parquet+ZSTD (~5×) |
|---|---|---|---|
| `trades_1s` (41 cols) | ~10 GB | ~1 TB | ~200 GB |
| `book_1s` (51 cols) | ~13 GB | ~1.3 TB | ~260 GB |
| **Total** | **~23 GB** | **~2.3 TB** | **~460 GB** |

## Practical examples

| Config | Disk/year (compressed) |
|---|---|
| 10 symbols | ~45 GB |
| 30 symbols | ~140 GB |

## Directory

`CT_MCP_STREAMS_DIR` > `<exe_dir>/streams/` > tempdir fallback.

Parquet partitioned: `{streams_dir}/trades_1s/{provider}/{symbol}/{YYYY-MM-DD}.parquet`

---

> Next: [WebSocket gateway](./07-gateway.en.md)
