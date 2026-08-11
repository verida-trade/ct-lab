# Modos do CT Lab (Auto vs Chat)

O CT Lab tem dois modos de operação:

| Modo | Descrição | Quando usar |
|---|---|---|
| `auto` | O agente decide quando e quais tools chamar | Fluxo natural, hands-off |
| `chat` | O agente pergunta antes de cada ação | Controle total, debugging |

Configure via env: `CT_MODE=auto` ou `CT_MODE=chat`

> Veja [Instalação — Provider de IA](../01-instalacao/02-provider-ia.pt.md) para detalhes de configuração do provider.
