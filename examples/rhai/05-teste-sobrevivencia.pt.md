# Receita 05 — Teste de Sobrevivência (Grid)

> **Nível:** Intermediário · **Plano:** Premium
> **Pré-requisito:** Leitura de [`docs/08-doutrina/`](../../docs/08-doutrina/) — entender o conceito de "lado aleatório" e por que um sistema lucrativo precisa sobreviver ao piso do acaso.
> **Series source:** `ct://series/binance/BTCUSDT/15m` (1724 candles)

---

## O que é?

- O **Teste de Sobrevivência** dispara compra **e** venda no **mesmo** momento, com o **mesmo** gestor de risco. Ele mede o quão negativo é o piso de escolha arbitrária de lado.
- Se o sistema não tem _edge_ de entrada, o resultado médio dos pares (compra + venda) deve tender a zero **antes** de custos. Qualquer valor negativo é "sangramento" puro da execução.
- A métrica-chave é o **EV_par_reguas**: o valor esperado de cada par **normalizado pela régua** (distância stop→ ativação). `EV ≈ 0` significa piso neutro; `EV ≪ 0` significa sangramento.
- É o teste mais cruel que existe: ele não pergunta "você acertou o lado?" — ele pergunta "quanto você perde só por _entrar_?".

---

## Passo 1 — Buscar série

```json
{ "name": "buscar_binance", "arguments": { "par": "BTCUSDT", "tempo": "15m" } }
```

A série fica disponível em `ct://series/binance/BTCUSDT/15m`.

---

## Passo 2 — (Opcional) Medir estrutura

```json
{ "name": "ct_medir_estrutura", "arguments": { "serie": "ct://series/binance/BTCUSDT/15m" } }
```

Retorna a régua de volatilidade típica do ato, útil para calibrar `stop_r` e `dist_r`.

---

## Passo 3 — Rodar o teste

```json
{
  "name": "ct_testar_sobrevivencia",
  "arguments": {
    "serie": "ct://series/binance/BTCUSDT/15m",
    "momentos": 20,
    "fee_pct": 0,
    "stop_r": 0.5,
    "ativacao_r": 1.0,
    "dist_r": 0.3,
    "prazo": 128,
    "breakeven": true,
    "reescala_vol": true,
    "piramide": false
  }
}
```

| Parâmetro      | Valor | Significado                                     |
|----------------|-------|-------------------------------------------------|
| `momentos`     | 20    | Quantos pontos no tempo disparam o par compra+venda |
| `fee_pct`      | 0     | taxa percentual por trade (0 = sem custo)       |
| `stop_r`       | 0.5   | stop em múltiplos da régua                       |
| `ativacao_r`   | 1.0   | ativação (alvo) em múltiplos da régua            |
| `dist_r`       | 0.3   | distância de entrada em múltiplos da régua       |
| `prazo`        | 128   | candles máximos por trade                        |
| `breakeven`    | true  | move stop para ponto neutro após ativar          |
| `reescala_vol` | true  | normaliza stop/alvo pela volatilidade local      |
| `piramide`     | false | sem adicionar lotes                             |

A resposta contém `n_momentos` entradas, cada uma com: `ts`, `regua`, `pnl_compra`, `pnl_venda`, `par_reguas`.

---

## Resultado real — 3 variantes

Rodamos três variantes sobre a mesma série (BTCUSDT 15m, 1724 candles, 20 momentos):

| Variante             | fee_pct | piramide | soma_pnl      | soma_par_reguas | EV_par_reguas | pares_positivos |
|----------------------|---------|----------|---------------|-----------------|---------------|-----------------|
| ① Sem taxa, sem pir. | 0       | false    | **−$1 004,03**| −0,777          | **−0,0388**   | **12 / 20**      |
| ② Com taxa 0,1 %     | 0.1     | false    | −$6 132,61    | −8,13           | −0,407        | 4 / 20           |
| ③ Sem taxa, com pir. | 0       | true     | −$2 311,48    | −3,43           | −0,171        | 11 / 20          |

**Exemplos de momentos (variante ①):**

| ts          | régua    | pnl_compra | pnl_venda | par_reguas |
|-------------|----------|------------|-----------|------------|
| 1785447900  | 1233,31  | 0,00       | −616,65   | −0,500     |
| 1785491100  | 1771,38  | −899,79    | 1007,85   | +0,061     |
| 1785753900  | 1496,33  | +997,38    | −748,17   | +0,167     |

---

## Interpretação

- **EV_par_reguas = −0,0388 (sem taxa)** → cada par perde ~3,9 % da régua. É um viés pequeno, mas **negativo** — a execução sangra, mesmo que pouco, da escolha arbitrária de lado.
- **12 / 20 pares positivos (sem taxa)** → o piso é _marginal_. A execução não sangra muito: em 60 % dos momentos, um par compra+venda fecha no verde. Isso significa que o _edge_ precisa vir da **entrada**, não da execução.
- **Com taxa 0,1 %: só 4 / 20 pares positivos, EV = −0,407** → catastrófico. A taxa transforma 12/20 em 4/20. O custo de transação destrói o piso neutro. Conclusão: o _setup_ tem que produzir _edge_ via entrada, não via execução.
- **Pirâmide piora: −$2 311 vs −$1 004** → adicionar tamanho não melhora a relação _edge_/_custo_. Pirâmide é alavancagem sobre um piso que já é marginal.
- **Escala dos PnLs:** a régua varia de ~1 233 a ~1 771 (volatilidade local reescalada). A soma de $1 004 de prejuízo sobre 20 pares é ~$50/par — objeto pequeno, mas consistente.
- **Sentença:** o Teste de Sobrevivência não diz _se_ você ganha dinheiro; diz _quanto_ o simples ato de entrar custa. Se o piso é negativo, qualquer estratégia precisa de um _edge_ de entrada maior que o sangramento medido aqui.

---

## Variações

- 🔧 **Trocar `stop_r` / `ativacao_r` / `dist_r`** →Uma régua mais larga (`stop_r = 1.0`) reduz o custo relativo da taxa; uma ativação mais curta (`ativacao_r = 0.5`) aproxima a saída e reduz a exposição ao piso.
- 📈 **`piramide = true`** → Reveja se o seu _setup_ tem _edge positivo_ comprovado; pirâmide só faz sentido quando o piso já é positivo.
- ⏱️ **Outro _timeframe_** (`5m`, `1h`) → Altera a régua típica e o número de momentos úteis. Séries mais longas tendem a reduzir o ruído do piso.
- 🔢 **Mais momentos (`momentos = 50`)** → Reduz a variância da estimativa de EV; recomendado antes de declarar que um piso é "real".
