# Pipeline Declarativo (DAG)

> `montar_pipeline_indicadores` — encadeie indicadores e ops em um grafo acíclico (DAG) declarativo. A forma mais poderosa de compor sinais.

A pipeline permite executar uma sequência de steps onde cada step referencia steps anteriores via `$id`. Só o step final (`output`) é persistido como série derived. Intermediários ficam em memória — não poluem o cache.

---

## Anatomia de uma pipeline

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "meu_sinal",
    "output": "$sinal",
    "steps": [ ... ]
  }
}
```

| Campo | O que é |
|---|---|
| `anchor` | URI raw que ancora toda a pipeline. Define a linha do tempo. |
| `name` | Nome da série final: `ct://derived/<name>` |
| `output` | Referência `$<id>` ao step cujo output vira a série persistida |
| `steps` | Lista de steps em ordem topológica |

Cada step tem:
- `id` — identificador único (referenciado por outros steps via `$id`)
- `operacao` — o que fazer (indicador, combinar, transformar, etc.)
- `source` — entrada: `$anchor`, `$<id>` (step anterior), ou uma URI

---

## Ops disponíveis

Consulte sempre o catálogo vivo:

**Resource:** `ct://pipeline/catalog`

### Indicadores (mesmas 53 tools)
Cada indicador é um op na pipeline. Exemplo: `{ "id": "meu_rsi", "operacao": "rsi", "source": "$anchor", "period": 14 }`

### Ops declarativas

| Op | O que faz | Exemplo |
|---|---|---|
| `combinar_aritmetica` | `+ − × ÷` entre N parcelas (séries ou escalares) | Somar RSI + MFI |
| `comparar` | Relação/cruzamento série×série → 0/1 | `cruza_acima`, `cruza_abaixo`, `maior`, `menor` |
| `condicional` | Ternário: se condição → então X senão Y | `if rsi > 30 then 1 else 0` |
| `transformar` | `abs`, `log`, `sqrt`, `clamp`, `sinal`, `neg` | Log do volume |
| `estatistica_rolling` | `rma`, `smm`, `desvio_padrao`, `regressao_linear` janela móvel | Desvio padrão de 20 bars |
| `compose` | Inner-join de steps por timestamp (cross-series) | BTC + ETH alinhados |
| `custom` | Script Rhai ou Python inline/uri | Lógica customizada |

> **Caveat:** `comparar` é **série×série**. Threshold contra escalar (`rsi < 30`) é `condicional` ou `custom`, não `comparar`.

---

## Exemplo completo: Sinal de cruzamento RSI×SMA

```json
{
  "name": "montar_pipeline_indicadores",
  "arguments": {
    "anchor": "ct://series/binance/BTCUSDT/15m",
    "name": "sinal_rsi_sma",
    "output": "$sinal",
    "steps": [
      { "id": "rsi", "operacao": "rsi", "source": "$anchor", "period": 14 },
      { "id": "sma_rsi", "operacao": "sma", "source": "$rsi", "period": 9 },
      {
        "id": "cruz_acima",
        "operacao": "comparar",
        "esquerda": "$rsi",
        "direita": "$sma_rsi",
        "operador": "cruza_acima"
      },
      {
        "id": "cruz_abaixo",
        "operacao": "comparar",
        "esquerda": "$rsi",
        "direita": "$sma_rsi",
        "operador": "cruza_abaixo"
      },
      {
        "id": "sinal",
        "operacao": "condicional",
        "condicao": "$cruz_acima",
        "entao": { "escalar": 1.0 },
        "senao": { "escalar": -1.0 },
        "coluna_saida": "sinal"
      }
    ]
  }
}
```

**Retorno:**
```json
{
  "uri": "ct://derived/sinal_rsi_sma",
  "anchor_uri": "ct://series/binance/BTCUSDT/15m",
  "row_count": 1000,
  "first_ts": 1784060100,
  "last_ts": 1784951100,
  "steps_executed": 5
}
```

---

## Exemplo: Filtro ADX + DI

```json
{
  "anchor": "ct://series/binance/BTCUSDT/15m",
  "name": "filtro_tendencia",
  "output": "$filtro",
  "steps": [
    { "id": "adx", "operacao": "adx", "source": "$anchor" },
    {
      "id": "di_pos",
      "operacao": "transformar",
      "source": "$adx",
      "funcao": "sinal"
    },
    {
      "id": "adx_forte",
      "operacao": "condicional",
      "condicao": "$adx",
      "coluna_condicao": "adx",
      "entao": { "escalar": 1.0 },
      "senao": { "escalar": 0.0 },
      "coluna_saida": "tendencia_forte"
    },
    {
      "id": "filtro",
      "operacao": "comparar",
      "esquerda": "$adx",
      "coluna_esquerda": "adx",
      "direita": { "escalar": 25.0 },
      "operador": "maior"
    }
  ]
}
```

---

## Quando usar pipeline vs Rhai vs tool direta

| Via | Quando usar |
|---|---|
| **Tool direta** (`sma`, `rsi`, etc.) | 1 indicador simples, sem combinação |
| **Rhai vetorizado** (`materializar_indicador`) | Expressão única que encadeia indicadores: `sma(rsi(close, 14), 5)` |
| **Pipeline (DAG)** | 4+ steps, lógica declarativa reutilizável, múltiplos ramos |
| **Compose** (`compor_serie`) | Juntar N séries de **ativos diferentes** por timestamp |

> **Regra:** pipeline é a única via que suporta árvores (vários ramos convergindo). Rhai é linear (uma expressão). Tool direta é um cálculo só.

---

> Próximo: [Ops declarativas — detalhes](./04-ops-declarativas.pt.md) · [Rhai vetorizado](./05-rhai-vetorizado.pt.md) · [Cookbook](./07-cookbook.pt.md)
