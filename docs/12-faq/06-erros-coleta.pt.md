# Erros de Coleta

> Problemas comuns com coleta de microestrutura (book, trades, klines).

## WebSocket desconecta frequentemente

**Causa:** Instabilidade de rede, firewall bloqueando WebSocket, ou rate limit da Binance.

**Solução:**
1. Verifique se a conexão está estável
2. Reduza o número de símbolos simultâneos (recomendado: máx 20-50 símbolos)
3. Verifique se há firewall/proxy bloqueando `wss://stream.binance.com`
4. O coletor faz **reconnect automático** — se persistir, reinicie a tarefa:
```python
parar_coleta(id="minha_coleta")
criar_tarefa_coleta(symbols=["BTCUSDT"], tipos=["trades"])
```

## Coletor em estado "Failed"

**Causa:** Erro irrecuperável — geralmente URL de WS inválida, provider indisponível, ou permissão de escrita no diretório de streams.

**Solução:**
1. Verifique o erro detalhado no status:
```python
listar_tarefas()  # mostra estado e última mensagem de erro
```
2. Se for permissão: verifique o diretório de Parquet:
```bash
# Env var para override do diretório de streams
export CT_MCP_STREAMS_DIR=/path/to/streams
```
3. Pare e recrie a tarefa:
```python
parar_coleta(id="tarefa_falhada")
criar_tarefa_coleta(symbols=["BTCUSDT"], tipos=["book"])
```

## "No data in the last N seconds"

**Causa:** O coletor está conectado mas não recebe mensagens. Pode ser símbolo inativo ou market data desativado.

**Solução:**
1. Verifique se o símbolo está ativo na Binance
2. Alguns símbolos têm baixa liquidez — normal em pares pequenos
3. Para book: certifique-se de que o símbolo tem order book ativo

## Timestamps inconsistentes (ms vs μs)

**Causa:** A Binance usa timestamps em **microssegundos** a partir de 2025-01-01 e **milissegundos** antes. O ct-mcp-server normaliza automaticamente.

**Solução:** Normalmente transparente. Se você usa os dados via Parquet exportado:
- Antes de 2025-01-01: timestamps em ms (13 dígitos)
- A partir de 2025-01-01: timestamps em μs (16 dígitos)

## Backfill de trades: "bulk dump limit exceeded"

**Causa:** A Binance limita bulk dumps (downloads em massa) a um período máximo. O default é 7 dias (`backfill_dias: 7`).

**Solução:**
1. Reduza `backfill_dias`:
```python
criar_tarefa_coleta(
    symbols=["BTCUSDT"],
    tipos=["trades"],
    backfill_dias=3  # reduzido de 7 para 3
)
```
2. Para períodos maiores, use `buscar_serie_historico` (baixa em chunks sucessivos)

## Klines não coletam em tempo real

**Causa:** Klines são coletados via REST (poll) em vez de WebSocket. Se o intervalo de polling for muito longo, pode haver gap.

**Solução:**
1. Klines são coletados via `coletar_klines` (REST polling)
2. O intervalo padrão é adequado para a maioria dos casos
3. Para tempo real de candles, considere usar coleta de trades e construir klines via pipeline

## Limite de símbolos em coleta simultânea

**Causa:** Cada símbolo abre 1-3 conexões WebSocket. Muitos símbolos podem sobrecarregar CPU/RAM.

**Recomendação:**
- ≤ 20 símbolos: funcionamento suave
- 20-50 símbolos: monitorar CPU
- 50-100 símbolos: apenas em máquinas potentes (16GB+ RAM)
- > 100 símbolos: não recomendado

> Voltar para: [README](./README.pt.md)
