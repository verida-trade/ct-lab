# 01 — Guia Rápido da API

O **CT Lab Desktop** executa internamente um servidor HTTP (`ct-labd`) na
porta `8420` (configurável). Esse servidor expõe uma **API REST** que permite
acionar receitas, gerenciar agentes, agendar jobs em cron, trocar prompts e
enviar mensagens de chat — tudo programaticamente a partir de aplicações
externas (scripts, automações, CI/CD, bots, etc.).

Este guia é prático: em poucos minutos você fará seu primeiro request e
entenderá os principais fluxos. Para a referência completa de rotas, consulte
o código-fonte do servidor (`ct-labd`).

---

## Sumário

- [Pré-requisitos](#pré-requisitos)
- [Onde obter a secret key](#onde-obter-a-secret-key)
- [Autenticação](#autenticação)
- [Health check](#health-check)
- [Primeiro request: chat via `/reply`](#primeiro-request-chat-via-reply)
- [Iniciar agente com receita](#iniciar-agente-com-receita)
- [Listar receitas locais](#listar-receitas-locais)
- [Ler recurso MCP diretamente](#ler-recurso-mcp-diretamente)
- [Criar e disparar job agendado](#criar-e-disparar-job-agendado)
- [Exemplo completo em Python](#exemplo-completo-em-python)
- [Exemplo completo em JavaScript/Node](#exemplo-completo-em-javascriptnode)
- [Tunnel: acesso remoto](#tunnel-acesso-remoto)
- [Troubleshooting](#troubleshooting)

---

## Pré-requisitos

| Item | Detalhe |
|------|---------|
| CT Lab Desktop | Versão com `ct-labd` habilitado (verifique em **Settings → App**) |
| Porta local | `8420` por padrão — pode ser alterada em **Settings → App → API Access** |
| Secret Key | Token gerado em **Settings → App → API Access** |
| `curl` | Útil para testes rápidos do terminal |
| Python ≥ 3.8 | Para os exemplos em Python (`requests`, `sseclient-py`) |
| Node.js ≥ 18 | Para os exemplos em JavaScript (fetch nativo, `EventSource`) |

> 💡 **Dica**: O servidor só aceita conexões de `127.0.0.1` por padrão. Para
> acesso remoto, use o [tunnel](#tunnel-acesso-remoto).

---

## Onde obter a secret key

A autenticação da API usa uma **secret key** estática gerada pelo CT Lab. Para
obtê-la:

1. Abra o **CT Lab Desktop**.
2. Navegue até **Settings → App → API Access**.
3. Se ainda não houver uma chave, clique em **Generate**.
4. **Copie o valor** — ele será mostrado apenas uma vez.
5. Confirme que **Enable API server** está ativado.

> ⚠️ **Trate a secret key como uma senha.** Nunca a comite em repositórios
> públicos. Use variáveis de ambiente (`CT_API_KEY`) ou um cofre de segredos.

---

## Autenticação

Toda requisição (exceto [`/status`](#health-check) e `/features`) deve incluir
o header:

```
X-Secret-Key: <sua-secret-key>
```

Sem o header correto, o servidor responde **401 Unauthorized**.

### Variável de ambiente recomendada

```bash
# Exporte uma vez por shell session
export CT_API_KEY="sua-secret-key-aqui"
export CT_API_URL="http://127.0.0.1:8420"
```

A partir de agora, todos os exemplos assumem que essas duas variáveis estão
definidas.

---

## Health check

Antes de qualquer chamada autenticada, verifique se o servidor está no ar:

```bash
# /status não exige o header X-Secret-Key
curl -s http://127.0.0.1:8420/status | jq .
```

Resposta esperada (exemplo):

```json
{
  "status": "ok",
  "version": "1.4.2",
  "uptime_seconds": 3600
}
```

> ℹ️ `/features` também é público e retorna as features habilitadas (premium,
> extensões, etc.).

---

## Primeiro request: chat via `/reply`

O endpoint `POST /reply` envia uma mensagem de usuário e retorna um **stream
SSE** (`text/event-stream`) com a resposta da IA em tempo real.

### Estrutura do body

```json
{
  "session_id": "minha-sessao-01",
  "user_message": {
    "role": "user",
    "content": [
      { "type": "text", "text": "Busque BTCUSDT em 15m e calcule RSI(14)" }
    ]
  },
  "recipe_name": null,
  "override_conversation": false
}
```

### curl (stream ao vivo)

```bash
# O -N desabilita buffer para ver o stream em tempo real
curl -N \
  -X POST "$CT_API_URL/reply" \
  -H "X-Secret-Key: $CT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "minha-sessao-01",
    "user_message": {
      "role": "user",
      "content": [
        { "type": "text", "text": "Busque BTCUSDT em 15m e calcule RSI(14)" }
      ]
    }
  }'
```

### Eventos SSE

Cada evento é uma linha `data: {"type":"...", ...}\n\n`. Os principais tipos:

| `type` | O que representa |
|--------|------------------|
| `Message` | Um pedaço (token) da resposta da IA |
| `Finish` | Fim do stream — contém `reason` (ex.: `"stop"`) |
| `Notification` | Mensagem/notificação do agente |
| `UpdateConversation` | Snapshot da conversa atualizada |
| `ActiveRequests` | IDs de requests em andamento |
| `Error` | Erro — contém campo `error` |
| `Ping` | Keepalive |

Exemplo bruto:

```
data: {"type":"Message","message":{"role":"assistant","content":[{"type":"text","text":"Buscando"}]},"token_state":{"input":12,"output":3}}

data: {"type":"Message","message":{"role":"assistant","content":[{"type":"text","text":" BTCUSDT..."}]},"token_state":{"input":12,"output":8}}

data: {"type":"Finish","reason":"stop","token_state":{"input":12,"output":42}}
```

---

## Iniciar agente com receita

Para rodar uma receita como agente autônomo:

```bash
# 1. Inicia o agente com uma receita existente
curl -s -X POST "$CT_API_URL/agent/start" \
  -H "X-Secret-Key: $CT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "working_dir": "/home/user/projeto",
    "recipe_name": "rsi-diario-btc",
    "session_type": "plan_and_execute"
  }' | jq .
```

A resposta inclui `session_id` — guarde-o para as próximas chamadas:

```json
{
  "session_id": "agent-abc123",
  "status": "started"
}
```

### Outras operações com a sessão

```bash
# Listar ferramentas MCP disponíveis para a sessão
curl -s "$CT_API_URL/agent/tools?session_id=agent-abc123&extension_name=developer" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .

# Parar o agente
curl -s -X POST "$CT_API_URL/agent/stop" \
  -H "X-Secret-Key: $CT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "session_id": "agent-abc123" }' | jq .

# Trocar o provedor/modelo em tempo de execução
curl -s -X POST "$CT_API_URL/agent/update_provider" \
  -H "X-Secret-Key: $CT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "agent-abc123",
    "provider": "anthropic",
    "model": "claude-sonnet-4-20250514"
  }' | jq .
```

---

## Listar receitas locais

```bash
curl -s "$CT_API_URL/recipes/list" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .
```

Resposta (exemplo):

```json
{
  "recipes": [
    {
      "id": "rsi-diario-btc",
      "title": "RSI Diário BTC",
      "description": "Busca BTCUSDT e calcula RSI",
      "author": { "contact": "user@example.com" }
    }
  ]
}
```

---

## Ler recurso MCP diretamente

O endpoint `POST /agent/read_resource` lê um recurso MCP sem passar pela IA
(by-pass do modelo). Útil para scripts que querem dados estruturados rapidamente:

```bash
curl -s -X POST "$CT_API_URL/agent/read_resource" \
  -H "X-Secret-Key: $CT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "agent-abc123",
    "extension_name": "developer",
    "uri": "file:///home/user/projeto/README.md"
  }' | jq .
```

> 💡 Para listar recursos disponíveis, use `GET /agent/tools` com o filtro
> `extension_name` e procure por entradas do tipo `resource`.

---

## Criar e disparar job agendado

```bash
# 1. Cria um job com cron expression (roda todo dia às 09:00)
curl -s -X POST "$CT_API_URL/schedule/create" \
  -H "X-Secret-Key: $CT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "rsi-manha",
    "recipe": "rsi-diario-btc",
    "cron": "0 9 * * *"
  }' | jq .

# 2. Dispara imediatamente (não espera o próximo cron tick)
curl -s -X POST "$CT_API_URL/schedule/rsi-manha/run_now" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .
# → { "session_id": "job-session-xyz" }

# 3. Lista todos os jobs
curl -s "$CT_API_URL/schedule/list" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .

# 4. Pausa / reativa
curl -s -X POST "$CT_API_URL/schedule/rsi-manha/pause" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .

curl -s -X POST "$CT_API_URL/schedule/rsi-manha/unpause" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .

# 5. Remove
curl -s -X DELETE "$CT_API_URL/schedule/delete/rsi-manha" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .
```

---

## Exemplo completo em Python

Requer `pip install requests sseclient-py`.

```python
import os
import json
import requests

# Configuração
API_URL = os.environ["CT_API_URL"]            # http://127.0.0.1:8420
API_KEY = os.environ["CT_API_KEY"]            # sua secret key
HEADERS = {
    "X-Secret-Key": API_KEY,
    "Content-Type": "application/json",
}

# --- 1. Health check ---
r = requests.get(f"{API_URL}/status")
print("status:", r.json())

# --- 2. Listar receitas ---
r = requests.get(f"{API_URL}/recipes/list", headers=HEADERS)
for recipe in r.json().get("recipes", []):
    print("-", recipe["id"], "—", recipe.get("title"))

# --- 3. Iniciar agente ---
r = requests.post(f"{API_URL}/agent/start", headers=HEADERS, json={
    "working_dir": os.getcwd(),
    "recipe_name": "rsi-diario-btc",
})
session_id = r.json()["session_id"]
print("session_id:", session_id)

# --- 4. Chat via /reply com stream SSE ---
from sseclient import SSEClient

def stream_reply(session_id: str, message: str):
    """Envia uma mensagem e imprime os tokens SSE em tempo real."""
    body = {
        "session_id": session_id,
        "user_message": {
            "role": "user",
            "content": [{"type": "text", "text": message}],
        },
    }
    # stream=True + SSEClient para ler o event-stream incrementalmente
    resp = requests.post(
        f"{API_URL}/reply",
        headers=HEADERS,
        json=body,
        stream=True,
    )
    resp.raise_for_status()

    client = SSEClient(resp)
    for event in client.events():
        data = json.loads(event.data)
        t = data.get("type")
        if t == "Message":
            # Concatena o texto de cada pedaço da resposta
            for part in data["message"].get("content", []):
                if part.get("type") == "text":
                    print(part["text"], end="", flush=True)
        elif t == "Finish":
            print("\n[finish]", data.get("reason"))
            break
        elif t == "Error":
            print("\n[error]", data.get("error"))
            break

stream_reply(session_id, "Busque BTCUSDT em 15m e calcule RSI(14)")

# --- 5. Para o agente ao final ---
requests.post(
    f"{API_URL}/agent/stop",
    headers=HEADERS,
    json={"session_id": session_id},
)
print("agente parado")
```

---

## Exemplo completo em JavaScript/Node

Usa `fetch` nativo (Node ≥ 18) e `EventSource` do pacote `eventsource`.

```bash
npm install eventsource
```

```javascript
// api-quickstart.mjs
import EventSource from "eventsource";

const API_URL = process.env.CT_API_URL ?? "http://127.0.0.1:8420";
const API_KEY = process.env.CT_API_KEY;

if (!API_KEY) {
  console.error("Defina CT_API_KEY no ambiente.");
  process.exit(1);
}

const headers = {
  "X-Secret-Key": API_KEY,
  "Content-Type": "application/json",
};

// --- 1. Health check ---
const status = await (await fetch(`${API_URL}/status`)).json();
console.log("status:", status);

// --- 2. Listar receitas ---
const recipesResp = await fetch(`${API_URL}/recipes/list`, { headers });
const { recipes } = await recipesResp.json();
recipes.forEach((r) => console.log("-", r.id, "—", r.title));

// --- 3. Iniciar agente ---
const startResp = await fetch(`${API_URL}/agent/start`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    working_dir: process.cwd(),
    recipe_name: "rsi-diario-btc",
}),
});
const { session_id } = await startResp.json();
console.log("session_id:", session_id);

// --- 4. Chat via /reply com stream SSE ---
// EventSource não suporta POST nem headers custom,
// então usamos fetch manual com um ReadableStream reader.
async function streamReply(sessionId, message) {
  const resp = await fetch(`${API_URL}/reply`, {
    method: "POST",
    headers,
    body: JSON.stringify({
      session_id: sessionId,
      user_message: {
        role: "user",
        content: [{ type: "text", text: message }],
      },
    }),
  });

  const reader = resp.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  // Lê o stream incrementalmente
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    // SSE: eventos separados por "\n\n"
    const parts = buffer.split("\n\n");
    buffer = parts.pop(); // mantém o último fragmento incompleto

    for (const part of parts) {
      const line = part.trim();
      if (!line.startsWith("data:")) continue;
      const evt = JSON.parse(line.slice(5).trim());

      if (evt.type === "Message") {
        for (const seg of evt.message?.content ?? []) {
          if (seg.type === "text") process.stdout.write(seg.text);
        }
      } else if (evt.type === "Finish") {
        console.log("\n[finish]", evt.reason);
        return;
      } else if (evt.type === "Error") {
        console.error("\n[error]", evt.error);
        return;
      }
    }
  }
}

await streamReply(session_id, "Busque BTCUSDT em 15m e calcule RSI(14)");

// --- 5. Para o agente ao final ---
await fetch(`${API_URL}/agent/stop`, {
  method: "POST",
  headers,
  body: JSON.stringify({ session_id }),
});
console.log("agente parado");
```

---

## Tunnel: acesso remoto

Por padrão o `ct-labd` só escuta em `127.0.0.1`. Para expor a API remotamente
(de uma VPS, CI/CD, etc.), use o **tunnel** embutido:

```bash
# Inicia o tunnel
curl -s -X POST "$CT_API_URL/tunnel/start" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .

# Consulta o status (retorna a URL pública + hostname + secret)
curl -s "$CT_API_URL/tunnel/status" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .
```

Resposta:

```json
{
  "state": "active",
  "url": "https://abc123.trycloudflare.com",
  "hostname": "abc123.trycloudflare.com",
  "secret": "tunnel-secret-xyz"
}
```

A partir de agora, use a URL pública como base e mantenha o header `X-Secret-Key`.

```bash
# Para o tunnel quando não precisar mais
curl -s -X POST "$CT_API_URL/tunnel/stop" \
  -H "X-Secret-Key: $CT_API_KEY" | jq .
```

> 🔒 **Segurança**: Mesmo com o tunnel ativo, toda requisição precisa do header
> `X-Secret-Key`. Nunca exponha a URL pública em logs ou canais públicos.

---

## Troubleshooting

### `401 Unauthorized`

| Causa provável | Solução |
|----------------|---------|
| Header `X-Secret-Key` ausente | Inclua o header em toda requisição |
| Secret key incorreta | Regenere em **Settings → App → API Access** |
| Servidor reiniciado e nova key gerada | Atualize a variável `CT_API_KEY` |

### `424 Failed Dependency`

| Causa provável | Solução |
|----------------|---------|
| Agente não inicializado para a sessão | Rode `/agent/start` antes de `/reply` |
| Extensão MCP desconectada | Verifique o status em **Settings → Extensions**; reinicie a sessão |
| Working dir inexistente | Confirme que o caminho em `working_dir` existe |

### Timeout / sem resposta no `/reply`

| Causa provável | Solução |
|----------------|---------|
| Provider de IA sem chave de API configurada | Configure em **Settings → AI Provider** |
| Rate limit do provedor | Reduza a frequência de chamadas; use backoff exponencial |
| Modelo muito lento para a tarefa | Troque para um modelo mais rápido via `/agent/update_provider` |
| Conexão fechada antes do `Finish` | Reenvie a mensagem; considere usar `override_conversation` |

### `ECONNREFUSED` (não conecta à porta 8420)

| Causa provável | Solução |
|----------------|---------|
| CT Lab Desktop não está rodando | Abra o aplicativo |
| Porta customizada diferente | Confirme em **Settings → App → API Access** o número da porta |
| Firewall bloqueando localhost | Libere a porta 8420 (ou a customizada) para `127.0.0.1` |

---

## Navegação

- ➡️ **[Voltar ao índice da API](./README.pt.md)**
- ⬅️ **[Voltar para Receitas](../11-receitas/README.pt.md)**

---

_Last updated: 2026-08-11_
