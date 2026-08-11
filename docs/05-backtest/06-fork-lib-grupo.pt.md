# Fork da Lib `grupo`

> **Premium.** Evolua a lib `grupo` do seed CT para a sua própria versão. A execução é SUA — faça a sua.

O seed CT fornece a lib `grupo` em `ct://libs/seed/grupo`. Você pode criar um fork com suas próprias regras e usá-la nos backtests.

---

## `salvar_lib`

```json
{
  "name": "salvar_lib",
  "arguments": {
    "name": "meu_grupo",
    "content": "// minha lib grupo customizada\n// ... veja o seed e modifique ..."
  }
}
```

A lib é persistida no servidor e pode ser importada nas estratégias como `import "meu_grupo" as g;`.

---

## `ler_lib` e `listar_libs`

```json
{ "name": "ler_lib", "arguments": { "name": "meu_grupo" } }
```

```json
{ "name": "listar_libs", "arguments": {} }
```

## `excluir_lib`

```json
{ "name": "excluir_lib", "arguments": { "name": "meu_grupo" } }
```

---

## Resolução (usuário-primeiro)

A estratégia importa a lib pelo nome. Se a lib do usuário existe, ela é usada; senão, o seed CT é usado como fallback.

---

## O que evitar

- Não use `comprado()`/`vendido()` por fora da lib na mesma estratégia que usa `grupo` no modo market.
- Sempre reatribua `estado = r.estado`.
- `estrategia` carrega estado entre backtests — recompile por backtest (não reuse a instância compilada).

---

> Próximo: [Camada adaptativa](./07-camada-adaptativa.pt.md) · [A lib `grupo`](./05-lib-grupo.pt.md)
