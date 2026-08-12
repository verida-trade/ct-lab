# Erros de Dados

> Problemas comuns com séries, URIs, cache e ingestão.

## "Series not found" / URI não resolve

```
Error: series not found: ct://series/binance/BTCUSDT/15m
```

**Causa:** A série nunca foi buscada ou foi removida do cache (LRU eviction no Premium).

**Solução:**
1. Verifique se a série está em cache:
```python
listar_series()  # lista todas as séries no cache
```
2. Se não estiver, busque novamente:
```python
buscar_serie(provider="binance", symbol="BTCUSDT", timeframe="15m")
```

## Series limit reached (1/1)

```
Error: series limit reached (1/1)
```

**Causa:** Plano Free permite apenas 1 série em cache.

**Solução:**
```python
# Remova a série atual antes de buscar outra
remover_serie(uri="ct://series/binance/BTCUSDT/15m")
buscar_serie(provider="binance", symbol="ETHUSDT", timeframe="15m")
```

Ou faça upgrade para Premium (100 séries simultâneas com LRU automático).

## "Column not found: close"

```
Error: column not found: close
```

**Causa:** A série não tem a coluna esperada. Pode ser uma série sintética (sem OHLCV) ou o nome da coluna é diferente.

**Solução:**
1. Liste as colunas disponíveis:
```python
info = info_serie(uri="ct://derived/meu_indicador")
# → mostra value_names e colunas disponíveis
```
2. Use o nome correto da coluna no lugar de `"close"`

## Erro de backfill: "bulk dump not available"

**Causa:** A Binance não disponibiliza bulk dumps para todos os timeframes ou o período solicitado é muito antigo.

**Solução:**
1. Use `buscar_serie_historico` (baixa em chunks em vez de bulk)
2. Reduza o período (ex.: 30 dias em vez de 1 ano)
3. Use timeframe maior (1h ou 1d em vez de 1m) — bulk dumps são mais disponíveis

## CSV importa mas dados ficam errados

**Causa:** Formato do CSV não corresponde ao esperado (colunas, separador, timestamp).

**Solução:**
1. O CSV deve ter colunas: `timestamp,open,high,low,close,volume`
2. Timestamp em **segundos** (Unix epoch), não milissegundos
3. Separador: vírgula (`,`)

```csv
timestamp,open,high,low,close,volume
1700000000,67100.5,67200.0,67050.0,67150.2,1234.5
1700003600,67150.2,67250.0,67100.0,67200.8,2345.6
```

## Indicador retorna NaN para as primeiras N barras

**Causa:** Indicadores precisam de um período mínimo ("warmup") antes de produzir valores válidos. Por exemplo, SMA(20) precisa de 20 barras para calcular a primeira média.

**Solução:** Isto é comportamento esperado, não um erro. Use `nz()` para substituir NaN por 0:
```rhai
nz(sma(close, 20))  // NaN → 0
```

Ou comece a leitura após o warmup:
```python
# Lê apenas as últimas 100 barras (pula warmup)
data = read_resource("ct://series/.../tail/100")
```

## "derived series already exists" ao materializar indicador

**Causa:** Já existe uma série derivada com o mesmo nome.

**Solução:**
1. Use um nome diferente (`name`):
```python
materializar_indicador(fonte="ct://series/...", name="btc_rsi_v2", receita="rsi(close, 14)")
```
2. Ou remova a série antiga primeiro:
```python
remover_serie(uri="ct://derived/btc_rsi")
```

> Voltar para: [README](./README.pt.md)
