# MCP Extensions

> How to add, configure, and manage MCP servers in CT Lab.

The `ct-mcp-server` ships with CT Lab Desktop and provides 120+ tools (data, indicators, backtest, ML, microstructure). The connection is configured under **Settings → Extensions** — see [Installation → MCP Connection](../01-instalacao/03-conexao-mcp.en.md) for the full walkthrough.

## Adding an MCP server

1. Go to **Settings** (`Cmd/Ctrl + ,`).
2. Open the **Extensions** tab.
3. Click **Add MCP Server**.
4. Fill in:

| Field | Value |
|-------|-------|
| **Name** | Unique name (e.g., `ct-mcp-server`) |
| **Type** | `stdio` (always — the server uses stdin/stdout) |
| **Binary path** | Full path to the binary |
| **Env vars** | AI provider vars (`CT_PROVIDER`, `CT_MODEL`, API keys) |

5. Click **Test** → should show `Connection successful — N tools available`.
6. Click **Save**.

## Multiple servers

CT Lab supports multiple MCP servers simultaneously. Each runs as an isolated subprocess with its own set of tools. This lets you combine `ct-mcp-server` with other community MCP servers.

## Restart and troubleshoot

| Action | How |
|--------|-----|
| Restart server | **Extensions → Restart** |
| Check status | Green/red indicator in Extensions |
| View logs | **Settings → Advanced → Open Logs** (subprocess stderr) |
| View fingerprint | MCP resource `ct://host/fingerprint` |

## Tool catalog

To list available tools from `ct-mcp-server`:

```json
{ "uri": "ct://catalog" }
```

Or via chat: *"List the available tools."*

> See also: [Environment Variables](./05-env-vars.en.md) for `ct-mcp-server` env vars.

---

> Next: [`.cthints` file](./04-cthints.en.md) · Back to: [README](./README.en.md)
