# 03 — Conexão MCP

Agora que você já tem o **CT Lab Desktop** instalado (passo 01) e um
**provedor de IA** configurado (passo 02), é hora de entender a conexão MCP.
O **ct-mcp-server** acompanha o CT Lab Desktop — não é necessária instalação
separada. Este documento mostra como o servidor se conecta ao CT Lab para
que a IA possa usar todas as ferramentas.

---

## Sumário

- [Como funciona a conexão](#como-funciona-a-conexão)
- [Conectar via interface (Settings)](#conectar-via-interface-settings)
- [O que acontece nos bastidores](#o-que-acontece-nos-bastidores)
- [Verificação da conexão](#verificação-da-conexão)
- [Problemas comuns](#problemas-comuns)
- [Status do servidor](#status-do-servidor)

---

## Como funciona a conexão

O CT Lab Desktop atua como o **host MCP** — ele gerencia o ciclo de vida do
`ct-mcp-server` que acompanha o aplicativo. Quando você envia uma mensagem
no chat, a sequência é:

```
  Você escreve: "Busque BTCUSDT 15m e calcule o RSI"
        │
        ▼
  ┌─ CT Lab Desktop ─────────────────────────────────┐
  │                                                   │
  │   1. Envia seu pedido para o provedor de IA       │
  │      (OpenAI / Anthropic / Google / Ollama)      │
  │                                                   │
  │   2. A IA decide chamar ferramentas MCP:          │
  │      • buscar_serie("BTCUSDT", "15m")             │
  │      • rsi(uri, period=14)                        │
  │                                                   │
  │   3. CT Lab repassa as chamadas via stdin para    │
  │      o subprocesso ct-mcp-server                  │
  │                                                   │
  │   4. ct-mcp-server executa e devolve resultado    │
  │      via stdout                                   │
  │                                                   │
  │   5. IA recebe o resultado e formula resposta     │
  │                                                   │
  └───────────────────────────────────────────────────┘
```

A IA nunca conversa diretamente com o `ct-mcp-server`. O CT Lab Desktop é o
intermediário: ele recebe as solicitações de ferramenta da IA, repassa-as ao
subprocesso e devolve as respostas.

---

## Conectar via interface (Settings)

Para vincular o ct-mcp-server ao CT Lab:

### Passo 1 — Abrir as extensões

1. Abra o CT Lab Desktop.
2. Vá em **Settings** (`Cmd/Ctrl + ,`).
3. Clique na aba **Extensions**.

### Passo 2 — Adicionar servidor MCP

1. Clique em **Add MCP Server**.
2. Preencha o formulário:

```
┌───────────────────────────────────────────────────────────────────────┐
│  Add MCP Server                                                       │
│                                                                       │
│  Name:             [ct-mcp-server          ]                          │
│  Type:             [stdio                  ▼]                        │
│  Binary path:     [~/ct-mcp/ct-mcp-server  ]                          │
│  Args:             [                        ]                          │
│  Env vars:         [CT_PROVIDER=openai      ]                          │
│                   [CT_MODEL=gpt-4o          ]                          │
│                   [OPENAI_API_KEY=sk-•••••  ]                          │
│                   [CT_MODE=auto             ]                          │
│                                                                       │
│           [Test]    [Cancel]    [Save]                                 │
└───────────────────────────────────────────────────────────────────────┘
```

| Campo | Valor |
|-------|-------|
| **Name** | `ct-mcp-server` (ou o nome que preferir) |
| **Type** | `stdio` (sempre — o servidor usa stdio, não rede) |
| **Binary path** | Caminho completo do binário (ex: `~/ct-mcp/ct-mcp-server`) |
| **Args** | Deixe vazio na maioria dos casos |
| **Env vars** | Variáveis de ambiente do provedor de IA (ver doc 02) |

3. Clique em **Test**. O CT Lab iniciará o subprocesso e verificará se os
   recursos básicos respondem.

> ✅ Se o teste passar, você verá: `Connection successful — 122 tools available`.

4. Clique em **Save**.

### Passo 3 — Confirmar o status

Na lista de extensões, você deve ver:

```
  ┌────────────────────────────────────────────────────────────────────┐
  │ Extensions                                                          │
  │                                                                    │
  │  ●  ct-mcp-server              stdio    Connected    122 tools      │
  │                                                                    │
  └────────────────────────────────────────────────────────────────────┘
```

O indicador verde **Connected** confirma que o subprocesso está ativo e
respondendo.

---

## O que acontece nos bastidores

Quando você salva a configuração, o CT Lab Desktop:

1. **Inicia o subprocesso** — o binário `~/ct-mcp/ct-mcp-server` é iniciado
   com as variáveis de ambiente configuradas.

2. **Estabelece o canal stdio** — a comunicação usa stdin/stdout (JSON-RPC
   sobre stdio). Não há portas de rede.

3. **Descobre as ferramentas** — o CT Lab consulta quais ferramentas o
   servidor expõe (ex: `buscar_serie`, `rsi`, `sma`, `ct_backtest`, etc.).

4. **Registra as ferramentas na IA** — o CT Lab envia a lista de ferramentas
   para o provedor de IA, para que este saiba o que pode chamar.

5. **Mantém o subprocesso ativo** — se o subprocesso terminar inesperadamente,
   o CT Lab o reinicia automaticamente.

```
  CT Lab Desktop
       │
       ├── spawn → ~/ct-mcp/ct-mcp-server (subprocesso stdio)
       │              │
       │              ├── stdin  ← JSON-RPC (chamadas de ferramenta)
       │              ├── stdout → JSON-RPC (respostas)
       │              └── stderr → logs
       │
       └── HTTP/WS → Provedor de IA (OpenAI / Anthropic / Google / Ollama)
```

---

## Verificação da conexão

### Teste rápido no chat

Abra o chat no CT Lab e digite:

> **"Liste as séries temporais disponíveis."**

O que acontece:

1. A IA recebe seu pedido.
2. A IA decide chamar a ferramenta `listar_series` (JSON-RPC) / `listarSeries` (TypeScript SDK).
3. O CT Lab repassa a chamada para o ct-mcp-server.
4. O servidor retorna as séries em cache (ou uma lista vazia se for a primeira vez).
5. A IA apresenta o resultado em linguagem natural.

**Resposta esperada:**

```text
🤖 A IA não encontrou séries em cache. Para buscar uma série, 
   basta me pedir — por exemplo: "Busque BTCUSDT em 15m na Binance."
```

### Teste avançado

Para confirmar que os dados de mercado estão acessíveis, digite:

> **"Busque BTCUSDT em 15m na Binance e me diga quantos candles foram carregados."**

A IA chamará `buscar_serie` (JSON-RPC) / `buscarSerie` (TypeScript SDK) com os
parâmetros, e o servidor buscará os dados da Binance:

```text
🤖 Busquei a série BTCUSDT em 15m na Binance.
   • URI: ct://series/binance/BTCUSDT/15m
   • Candles carregados: 500
   • Primeiro candle: 2026-08-10 00:00 UTC
   • Último candle: 2026-08-11 15:45 UTC
```

---

## Problemas comuns

| Problema | Causa provável | Solução |
|----------|---------------|---------|
| Status **Disconnected** | Caminho do binário incorreto | Verifique o campo **Binary path** em Extensions |
| `0 tools available` | O servidor iniciou mas não registrou ferramentas | Reinicie o servidor: clique em **Restart** em Extensions |
| IA não chama ferramentas | `CT_MODE` está como `chat` | Mude para `auto` em **Settings → AI Provider** |
| `Tool not found` | Servidor desatualizado | Atualize o CT Lab Desktop para a versão mais recente |
| `Connection timeout` | Provedor de IA sem resposta | Verifique a chave de API e a conexão de internet |
| Resource `ct://host/fingerprint` não responde | Servidor não conectado | Refaça a configuração em Extensions |

---

## Status do servidor

A qualquer momento, você pode verificar o status do servidor:

| Método | Como verificar |
|--------|----------------|
| Interface | **Settings → Extensions** — indicador verde/vermelho |
| Recurso MCP | `ct://host/fingerprint` — exibe a assinatura digital |
| Logs | **Settings → Advanced → Open Logs** — stderr do subprocesso |

> 💡 Se algo parar de funcionar, **Restart** em Extensions reinicia o
> subprocesso sem fechar o CT Lab.

---

## Próximos Passos

- ➡️ **[04 — Primeiro Projeto](./04-primeiro-projeto)** — Rode seu primeiro backtest de ponta a ponta
- ⬅️ **[02 — Provedor de IA](./02-provider-ia)** — Voltar para configuração da IA
- ⬅️ **[Índice](./README)**

---

_Last updated: 2026-08-11_
