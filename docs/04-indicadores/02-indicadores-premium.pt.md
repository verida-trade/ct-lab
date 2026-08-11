# Indicadores Premium (17 CT) — Requer Licença

> Indicadores proprietários da família CT. Visíveis e executáveis **apenas com licença premium válida**.

Os indicadores premium moram em uma camada separada no `ct-mcp-server`. Sem licença válida, eles não aparecem em `tools/list` e não podem ser chamados. A verificação acontece em duas camadas: estrutural (no startup do servidor) e per-call (a cada execução).

---

## Família CT — Balance/Flow/Imbalance

A família CT é baseada em um princípio: **imbalance é o primitivo**. Força = `(a − b) / (|a| + |b|)`, signed em `[-1, +1]`. Sem disputa → 0. Para cada indicador, há a versão bruta e a versão `ct_*` (VWMA windowed ponderada pelo size natural do indicador).

| Tool | Input | Janela | O que mede | Escala |
|---|---|---|---|---|
| `bop` | OHLC | — | Balance of Power (Livshin) — força do close vs open | `[-1, +1]` |
| `ct_bop` | OHLCV | `period` | VWMA do BOP ponderado por volume | `[-1, +1]` |
| `tfi` | `trades_1s` | — | Trade Flow Imbalance — `qty_delta / qty` do taker flow | `[-1, +1]` |
| `ct_tfi` | `trades_1s` | `period` | VWMA do TFI ponderado por `qty` | `[-1, +1]` |
| `bfi` | `book_1s` | — | Book Flow Imbalance — bid vs ask qty delta | `[-1, +1]` |
| `ct_bfi` | `book_1s` | `period` | VWMA do BFI ponderado por `\|bid_delta\| + \|ask_delta\|` | `[-1, +1]` |
| `obi` | `book_1s` | — | Order Book Imbalance top-of-book | `[-1, +1]` |
| `ct_obi` | `book_1s` | `period` | VWMA do OBI ponderado por `bid_qty + ask_qty` | `[-1, +1]` |

> **Nota:** `tfi`, `bfi`, `obi` e variantes `ct_*` exigem dados de microestrutura (`trades_1s` ou `book_1s`) coletados via `coletar_trades`/`coletar_book`. Veja [Microestrutura](../07-microestrutura/).

---

## Microestrutura do book (sem trades)

| Tool | Input | O que mede |
|---|---|---|
| `dbi_01` | `book_1s` | Depth Book Imbalance bin ±0.1% (versão profunda do OBI) |
| `dbi_1` | `book_1s` | Depth Book Imbalance bin ±1% |
| `mpo` | `book_1s` | Microprice Offset normalizado pelo half-spread; skew bullish ↔ `mpo > 0` |

---

## Análise estatística de candle

| Tool | Input | Parâmetros | O que produz |
|---|---|---|---|
| `ct_candle` | OHLCV | `period`, `short_period` | 28 features brutas candle-a-candle (direção/spread/volume + estruturais) |
| `ct_candle_classificado` | derived do `ct_candle` | 4 thresholds parametrizáveis | 18 eixos categóricos descritivos (sem nomenclatura técnica) |
| `ct_fibo_candle` | OHLCV | `period`, `threshold_volume_z` | Projeção Fibonacci a partir do range de candles âncora (30 colunas) |

### `ct_candle` — features produzidas

- **10 contínuas:** direção, corpo, pavio superior/inferior, spread, volume, etc.
- **7 comparativas (`cmp_*`):** ∈ {-1, 0, +1} — comparação com a barra anterior
- **4 booleanas (`is_*`):** ∈ {0, 1} — novos máximos/mínimos no período
- **7 estruturais:** `close_position_in_range`, `gap_pct_change`, `overlap_ratio`, `is_inside_bar`, `is_outside_bar`, `volume_slope_short`, `spread_slope_short`

### `ct_candle_classificado` — eixos categóricos

- 8 signed `{-1, 0, +1}`: direção/cmp/gap/tendências/aceleração
- 5 ordinais `0..N`: corpo/pavios/posição_close/regimes
- 3 categóricos: simetria_pavios, continuidade_direção, sobreposição
- 1 signed 5-state: divergência_esforço_resultado ∈ -2..+2

### `ct_fibo_candle` — o que retorna

30 colunas: `is_ancora` + 7 preços de nível Fibonacci (0/50/100/161.8/261.8/-61.8/-161.8) + 7 fases (ativo/invalido/alvo) + 7 direções + 7 defesas (pavio rejeitando nível) + `niveis_ativos`.

---

## Detectores proprietários (experimentais)

| Tool | Input | Parâmetros | O que detecta |
|---|---|---|---|
| `ct_swing` | OHLCV | `period_ct_bop`, `janela_calibracao`, `quantil_zero`, `quantil_extremo` | Virada de força via `ct_bop` com calibração rolante (quantis empíricos). 7 colunas incluindo `sinal_swing` ∈ {-2, -1, 0, +1, +2} |
| `ct_range` | OHLCV | `janela_zscore`, `janela_atr`, `k_mov`, `z_min`, `tau_r` | Detector de range a partir de candle âncora de alto volume. 4 colunas: `range_ativo`, `range_topo`, `range_fundo`, `range_idade` |
| `ct_tendencia` | OHLCV | mesmos params de `ct_range` | Estados causais de tendência pós-rompimento de range. 5 colunas: `tend_ativa`, `direcao`, `fase` (inativo/impulso/reteste/continuação/armadilha), `progresso`, `nivel_rompido` |
| `ct_momento` | OHLCV | params de range + `period` (default 20), `w_tercil` (default 480) | Momento CT com tercis rolling de volatilidade/volume |

### `ct_swing` — sinal_swing

| Valor | Significado |
|---|---|
| -2 | Reversão de alta (bearish) |
| -1 | Pullback de alta (bearish) |
| 0 | Nenhum sinal |
| +1 | Pullback de baixa (bullish) |
| +2 | Reversão de baixa (bullish) |

### `ct_tendencia` — fases

| Fase | Significado |
|---|---|
| 0 | Inativo |
| 1 | Impulso |
| 2 | Reteste |
| 3 | Continuação |
| 4 | Armadilha |

> **Importante:** "Posição não é sentença" — a fase 4 reporta regressão e pode recuperar. Rompimento pode virar armadilha → tendência oposta, ou vice-versa.

---

## Exemplo de uso

### `ct_candle` no BTCUSDT 15m

```json
{
  "name": "ct_candle",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "period": 20,
    "short_period": 5
  }
}
```

**Retorno (resumo):**
```json
{
  "uri": "ct://derived/ct_candle_...",
  "value_names": ["direcao", "corpo", "pavio_sup", "pavio_inf", "...28 colunas"],
  "valid_from_ts": 1784072400,
  "latest": [...]
}
```

### `ct_tendencia` com parâmetros custom

```json
{
  "name": "ct_tendencia",
  "arguments": {
    "uri": "ct://series/binance/BTCUSDT/15m",
    "janela_zscore": 96,
    "janela_atr": 14,
    "k_mov": 3,
    "z_min": 2.0,
    "tau_r": 1.5
  }
}
```

---

## Verificando licença

```json
// Resource: ct://license/info
// Retorna: { licensed: true, plan: "premium", expires_at: "2026-12-31", ... }
```

Se a licença expirar ou for inválida, as tools premium retornam erro com instrução de como renovar.

---

> Próximo: [Pipeline declarativo (DAG)](./03-pipeline-declarativo.pt.md) — combine indicadores
> Dados de microestrutura: [Coleta de trades/book](../07-microestrutura/)
