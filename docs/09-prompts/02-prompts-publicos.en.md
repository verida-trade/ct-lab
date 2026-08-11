# Public Prompts

> Available without a premium license.

| Prompt | What it does |
|---|---|
| `saudacao` | Onboarding — greets the user, introduces the platform |
| `comecar` | Guides new users: setup, first step, explore |
| `backtest` | Backtest workflow: fetch series → choose indicators → run → interpret |

## How to invoke

**Chat:**
> Use the prompt "backtest" with symbol=BTCUSDT interval=15m

**MCP:**
```json
{ "method": "prompts/get", "params": { "name": "backtest", "arguments": { "symbol": "BTCUSDT", "interval": "15m" } } }
```

---

> Next: [Private prompts](./03-prompts-privados.en.md)
