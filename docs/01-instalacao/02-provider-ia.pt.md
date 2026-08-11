# 02 — Provedor de IA

O CT Lab funciona com múltiplos provedores de IA. A IA é o "cérebro" que
interpreta seus pedidos em linguagem natural e decide quais ferramentas do
ct-mcp-server chamar. Este documento mostra como configurar cada provedor
suportado.

---

## Sumário

- [Visão geral](#visão-geral)
- [Variáveis de ambiente](#variáveis-de-ambiente)
- [OpenAI](#openai)
- [Anthropic](#anthropic)
- [Google AI](#google-ai)
- [Ollama (local)](#ollama-local)
- [Modos de operação (CT_MODE)](#modos-de-operação-ct_mode)
- [Configuração via interface](#configuração-via-interface)
- [Verificação](#verificação)
- [Troca de provedor](#troca-de-provedor)
- [Solução de problemas](#solução-de-problemas)

---

## Visão geral

O CT Lab suporta quatro provedores de IA:

| Provedor | Tipo | Requer internet | Modelos populares |
|----------|------|-----------------|-------------------|
| OpenAI | Cloud | ✅ | gpt-4o, gpt-4o-mini |
| Anthropic | Cloud | ✅ | claude-sonnet-4-20250514, claude-haiku |
| Google | Cloud | ✅ | gemini-2.0-flash, gemini-1.5-pro |
| Ollama | Local | ❌ | llama3, mistral, phi3, … |

> 💡 **Recomendado para iniciantes**: OpenAI gpt-4o ou Anthropic
> claude-sonnet-4-20250514 — ambos têm excelente suporte ao protocolo MCP.

> 🔒 **Privacidade total**: se você não quer enviar dados para a nuvem,
> use Ollama localmente.

---

## Variáveis de ambiente

Toda a configuração do provedor de IA pode ser feita via variáveis de ambiente
ou pela interface do CT Lab. As três variáveis principais são:

| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `CT_PROVIDER` | Identifica o provedor de IA | `openai`, `anthropic`, `google`, `ollama` |
| `CT_MODEL` | Especifica o modelo dentro do provedor | `gpt-4o`, `claude-sonnet-4-20250514`, `llama3` |
| `CT_MODE` | Modo de operação | `auto` (padrão) ou `chat` |

### Chaves de API

Cada provedor cloud exige uma chave de API:

| Variável | Provedor | Onde obter |
|----------|----------|------------|
| `OPENAI_API_KEY` | OpenAI | platform.openai.com/api-keys |
| `ANTHROPIC_API_KEY` | Anthropic | console.anthropic.com |
| `GOOGLE_API_KEY` | Google | aistudio.google.com/apikey |

> ⚠️ **Nunca compartilhe suas chaves de API.** O CT Lab armazena as chaves
> localmente e nunca as envia para servidores de terceiros.

---

## OpenAI

### Configuração por variáveis de ambiente

```bash
export CT_PROVIDER=openai
export CT_MODEL=gpt-4o
export OPENAI_API_KEY="sk-sua-chave-aqui"
export CT_MODE=auto
```

### Configuração via interface

1. Abra CT Lab → **Settings → AI Provider**.
2. Selecione **OpenAI** no menu suspenso.
3. Insira sua chave de API no campo **API Key**.
4. Selecione o modelo **gpt-4o** (ou outro disponível).
5. Defina o modo como **Auto** (recomendado).
6. Clique em **Save**.

### Verificar

```bash
# No terminal, antes de abrir o CT Lab:
echo $CT_PROVIDER   # deve imprimir: openai
echo $CT_MODEL       # deve imprimir: gpt-4o
```

---

## Anthropic

### Configuração por variáveis de ambiente

```bash
export CT_PROVIDER=anthropic
export CT_MODEL=claude-sonnet-4-20250514
export ANTHROPIC_API_KEY="sk-ant-sua-chave-aqui"
export CT_MODE=auto
```

### Configuração via interface

1. Abra CT Lab → **Settings → AI Provider**.
2. Selecione **Anthropic** no menu suspenso.
3. Insira sua chave de API no campo **API Key**.
4. Selecione o modelo **claude-sonnet-4-20250514** (ou outro disponível).
5. Defina o modo como **Auto** (recomendado).
6. Clique em **Save**.

### Verificar

```bash
echo $CT_PROVIDER   # deve imprimir: anthropic
echo $CT_MODEL       # deve imprimir: claude-sonnet-4-20250514
```

---

## Google AI

### Configuração por variáveis de ambiente

```bash
export CT_PROVIDER=google
export CT_MODEL=gemini-2.0-flash
export GOOGLE_API_KEY="sua-chave-aqui"
export CT_MODE=auto
```

### Configuração via interface

1. Abra CT Lab → **Settings → AI Provider**.
2. Selecione **Google** no menu suspenso.
3. Insira sua chave de API no campo **API Key**.
4. Selecione o modelo **gemini-2.0-flash** (ou outro disponível).
5. Defina o modo como **Auto** (recomendado).
6. Clique em **Save**.

### Verificar

```bash
echo $CT_PROVIDER   # deve imprimir: google
echo $CT_MODEL       # deve imprimir: gemini-2.0-flash
```

---

## Ollama (local)

Ollama permite rodar modelos de IA localmente, sem internet. É ideal para
privacidade total ou ambientes offline.

### Pré-requisitos

1. Instale o Ollama: [ollama.com](https://ollama.com)
2. Baixe um modelo:

```bash
ollama pull llama3
```

### Configuração por variáveis de ambiente

```bash
export CT_PROVIDER=ollama
export CT_MODEL=llama3
export CT_MODE=auto
```

> ⚠️ Não é necessária chave de API para Ollama local.

### Configuração via interface

1. Abra CT Lab → **Settings → AI Provider**.
2. Selecione **Ollama** no menu suspenso.
3. No campo **Model**, digite `llama3` (ou outro modelo já baixado).
4. Defina o modo como **Auto** (recomendado).
5. Clique em **Save**.

### Verificar

```bash
# Confirme que o Ollama está rodando
ollama list

# Deve listar:
# NAME       SIZE    MODIFIED
# llama3     4.7 GB  2 hours ago
```

> 💡 **Modelos recomendados para uso com MCP**: `llama3` (8B), `mistral` (7B),
> `phi3` (3.8B — mais leve). Modelos maiores (13B+) exigem mais RAM/GPU.

---

## Modos de operação (CT_MODE)

| Modo | Descrição | Quando usar |
|------|-----------|-------------|
| `auto` | A IA decide automaticamente quando e quais ferramentas chamar | ✅ Recomendado para a maioria dos casos |
| `chat` | A IA apenas conversa, sem chamar ferramentas automaticamente | Para discussões conceituais sem execução de ações |

Em modo `auto`, a IA:

1. Recebe seu pedido em linguagem natural.
2. Decide qual ferramenta MCP chamar (ex: `buscar_serie` / `buscarSerie`).
3. Executa a chamada via ct-mcp-server.
4. Analisa o resultado.
5. Chama mais ferramentas se necessário (ex: `rsi`, `ct_backtest` / `ctBacktest`).
6. Apresenta o resultado final em linguagem natural.

---

## Configuração via interface

Todas as variáveis de ambiente acima também podem ser definidas na interface:

```
┌───────────────────────────────────────────────────────────────────────┐
│  Settings > AI Provider                                               │
│                                                                       │
│  Provider:        [OpenAI    ▼]                                       │
│  Model:           [gpt-4o    ▼]                                       │
│  API Key:         [sk-•••••••••••••••••••••••••••••••••••]           │
│  Mode:            [Auto      ▼]                                       │
│                                                                       │
│           [Test Connection]    [Save]                                  │
└───────────────────────────────────────────────────────────────────────┘
```

Use o botão **Test Connection** para verificar se a IA responde corretamente
antes de salvar.

---

## Verificação

Após configurar, abra o chat no CT Lab e digite:

> "Olá! Liste as séries temporais disponíveis."

Se tudo estiver correto, a IA responderá com uma lista de séries — o que
significa que a conexão com o provedor e o ct-mcp-server está funcionando.

---

## Troca de provedor

Para trocar de provedor:

1. Vá em **Settings → AI Provider**.
2. Selecione o novo provedor.
3. Insira a nova chave de API (se aplicável).
4. Selecione o modelo.
5. Clique em **Test Connection** e depois **Save**.

> Não é necessário reiniciar o CT Lab. A mudança entra em vigor imediatamente.

---

## Solução de problemas

| Problema | Solução |
|----------|---------|
| `Authentication failed` | Verifique se a chave de API está correta e válida |
| `Model not found` | Confirme o nome exato do modelo (ex: `gpt-4o`, não `gpt4`) |
| Ollama não responde | Verifique se o serviço está ativo: `ollama serve` |
| Respostas sem ferramentas | Mude `CT_MODE` de `chat` para `auto` |
| Timeout | Verifique sua conexão com a internet (ou carga do Ollama local) |

---

## Próximos Passos

- ➡️ **[03 — Conexão MCP](./03-conexao-mcp)** — Conecte o ct-mcp-server ao CT Lab
- ⬅️ **[01 — CT Lab Desktop](./01-ct-lab-desktop)** — Volte para a instalação do app
- ⬅️ **[Índice](./README)**

---

_Last updated: 2026-08-11_
