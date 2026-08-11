# Prompts Públicos

> Disponíveis sem licença premium.

| Prompt | O que faz |
|---|---|
| `saudacao` | Onboarding — sauda o usuário, apresenta a plataforma |
| `comecar` | Guia o usuário novo: configuração, primeiro passo, explorar |
| `backtest` | Workflow de backtest: buscar série → escolher indicadores → rodar → interpretar |

## Como invocar

**Chat:**
> Use o prompt "backtest" com symbol=BTCUSDT interval=15m

**MCP:**
```json
{ "method": "prompts/get", "params": { "name": "backtest", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } } }
```

---

> Próximo: [Prompts privados](./03-prompts-privados.pt.md)
