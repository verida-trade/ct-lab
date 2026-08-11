# Erros de Licença

> Problemas comuns com licença premium.

## "Tool não aparece na lista"

**Causa:** Licença inválida, expirada ou não configurada.

**Solução:**

1. Verifique com `ct://license/info`
2. Se sem licença: use `comprar_premium` para adquirir
3. Se expirada: renove via `comprar_premium`

## "Fingerprint mismatch"

O fingerprint da máquina não corresponde ao da licença.

**Causa:** A licença é vinculada ao hardware. Mudança de hardware, VM clonada ou container pode causar mismatch.

**Solução:** Reative a licença no novo hardware.

## Cap de séries (1 no free, 100 no premium)

```
Erro: limite de séries atingido (1/1)
```

**Solução:** Remova séries não usadas (`remover_serie`) ou faça upgrade para premium (100 séries).

---

> Voltar para: [README](./README.pt.md)
