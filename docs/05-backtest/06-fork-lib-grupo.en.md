# Forking the `grupo` Lib

> **Premium.** Evolve the CT seed `grupo` lib into your own version. The execution is YOURS — make it your own.

The CT seed provides the `grupo` lib at `ct://libs/seed/grupo`. You can create a fork with your own rules and use it in backtests.

---

## `salvar_lib`

```json
{
  "name": "salvar_lib",
  "arguments": {
    "name": "my_grupo",
    "content": "// my custom grupo lib\n// ... see the seed and modify ..."
  }
}
```

The lib is persisted on the server and can be imported in strategies as `import "my_grupo" as g;`.

---

## `ler_lib` and `listar_libs`

```json
{ "name": "ler_lib", "arguments": { "name": "my_grupo" } }
```

```json
{ "name": "listar_libs", "arguments": {} }
```

## `excluir_lib`

```json
{ "name": "excluir_lib", "arguments": { "name": "my_grupo" } }
```

---

## Resolution (user-first)

The strategy imports the lib by name. If the user's lib exists, it's used; otherwise, the CT seed is used as fallback.

---

> Next: [Adaptive layer](./07-camada-adaptativa.en.md) · [The `grupo` lib](./05-lib-grupo.en.md)
