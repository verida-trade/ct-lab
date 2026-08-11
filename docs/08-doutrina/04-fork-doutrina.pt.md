# Fork da Doutrina

> A doutrina é SUA — evolua a sua. O seed CT é o ponto de partida; o usuário desenvolve a própria.

A doutrina segue o mesmo modelo de propriedade de prompts e libs:

- **Baseline CT** (imutável): seed embarcado, servido em `ct://doutrina`
- **Fork do usuário** (editável, persistido): ao salvar, cria artefato novo do usuário
- **Resolução** (usuário-primeiro): serve a versão ativa do usuário; sem ela, o seed CT

## Tools da doutrina

### `salvar_doutrina`

```json
{
  "name": "salvar_doutrina",
  "arguments": {
    "tema": "ml",
    "nome": "minha_doutrina_ml",
    "conteudo": "# Minha doutrina de ML\n..."
  }
}
```

### `listar_doutrinas`

```json
{ "name": "listar_doutrinas", "arguments": {} }
```

### `aplicar_doutrina`

Define qual versão está ativa (servida em `ct://doutrina/{tema}`):

```json
{
  "name": "aplicar_doutrina",
  "arguments": { "tema": "ml", "nome": "minha_doutrina_ml" }
}
```

### `excluir_doutrina`

```json
{
  "name": "excluir_doutrina",
  "arguments": { "tema": "ml", "nome": "minha_doutrina_ml" }
}
```

## Por que persistir no servidor

O **modelo** lê doutrina via resource MCP. Para que a doutrina do usuário guie o modelo, ela precisa ser servida pelo `ct_mcp_server`. É o mesmo padrão de `salvar_filtro`/`listar_filtros`.

Resultado: **até os prompts baseline passam a usar a doutrina evoluída do usuário** automaticamente, porque referenciam o resource.

---

> Próximo: [Prompts e doutrina](./05-prompts-doutrina.pt.md)
