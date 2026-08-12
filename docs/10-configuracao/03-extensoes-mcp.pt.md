# Extensões MCP

> Como adicionar, configurar e gerenciar servidores MCP no CT Lab.

O `ct-mcp-server` acompanha o CT Lab Desktop e oferece +120 ferramentas (dados, indicadores, backtest, ML, microestrutura). A conexão é configurada em **Settings → Extensions** — veja [Instalação → Conexão MCP](../01-instalacao/03-conexao-mcp.pt.md) para o passo a passo.

## Adicionar um servidor MCP

1. Vá em **Settings** (`Cmd/Ctrl + ,`).
2. Abra a aba **Extensions**.
3. Clique em **Add MCP Server**.
4. Preencha:

| Campo | Valor |
|-------|-------|
| **Name** | Nome único (ex.: `ct-mcp-server`) |
| **Type** | `stdio` (sempre — o servidor usa stdin/stdout) |
| **Binary path** | Caminho completo do binário |
| **Env vars** | Variáveis do provider de IA (`CT_PROVIDER`, `CT_MODEL`, chaves) |

5. Clique em **Test** → deve mostrar `Connection successful — N tools available`.
6. Clique em **Save**.

## Múltiplos servidores

O CT Lab suporta vários servidores MCP simultaneamente. Cada um é um subprocesso isolado com seu próprio conjunto de ferramentas. Isso permite combinar o `ct-mcp-server` com outros servidores MCP da comunidade.

## Reiniciar e diagnosticar

| Ação | Como |
|------|------|
| Reiniciar servidor | **Extensions → Restart** |
| Verificar status | Indicador verde/vermelho em Extensions |
| Ver logs | **Settings → Advanced → Open Logs** (stderr do subprocesso) |
| Ver fingerprint | Recurso MCP `ct://host/fingerprint` |

## catatólogo de ferramentas

Para listar as ferramentas disponíveis do `ct-mcp-server`:

```json
{ "uri": "ct://catalog" }
```

Ou pelo chat: *"Liste as ferramentas disponíveis."*

> Veja também: [Variáveis de Ambiente](./05-env-vars.pt.md) para as env vars do `ct-mcp-server`.

---

> Próximo: [Arquivo `.cthints`](./04-cthints.pt.md) · Voltar para: [README](./README.pt.md)
