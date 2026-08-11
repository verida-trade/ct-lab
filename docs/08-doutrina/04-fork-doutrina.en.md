# Forking the Doctrine

> The doctrine is YOURS — evolve your own. The CT seed is the starting point; the user develops their own.

The doctrine follows the same ownership model as prompts and libs:

- **CT baseline** (immutable): embedded seed, served at `ct://doutrina`
- **User fork** (editable, persisted): saving creates a new user artifact
- **Resolution** (user-first): serves the active user version; without it, the CT seed

## Doctrine tools

### `salvar_doutrina`

```json
{
  "name": "salvar_doutrina",
  "arguments": { "tema": "ml", "nome": "my_ml_doctrine", "conteudo": "# My ML doctrine\n..." }
}
```

### `listar_doutrinas`

```json
{ "name": "listar_doutrinas", "arguments": {} }
```

### `aplicar_doutrina`

Sets which version is active (served at `ct://doutrina/{tema}`):

```json
{ "name": "aplicar_doutrina", "arguments": { "tema": "ml", "nome": "my_ml_doctrine" } }
```

### `excluir_doutrina`

```json
{ "name": "excluir_doutrina", "arguments": { "tema": "ml", "nome": "my_ml_doctrine" } }
```

## Why persist on the server

The **model** reads doctrine via MCP resource. For the user's doctrine to guide the model, it must be served by `ct_mcp_server`. Same pattern as `salvar_filtro`/`listar_filtros`.

Result: **even baseline prompts use the user's evolved doctrine** automatically, because they reference the resource.

---

> Next: [Prompts & doctrine](./05-prompts-doutrina.en.md)
