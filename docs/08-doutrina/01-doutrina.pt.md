# A Doutrina CT — Método Sugerido (Seed v3)

> **Não é veredito; é o método CT — sugerido e forkável.** Nada aqui é imutável: o usuário troca o que quiser. Este é o ponto de partida, servido em `ct://doutrina`.

## Axiomas

1. **Epistemológico (a raiz).** O futuro é **incognoscível**. Um indicador é só uma feature do passado — representa o que já aconteceu, não prevê.
2. **Operacional (corolário).** **Sobrevivência antes de edge.** Como não há previsão e o erro é inevitável, o **risco é a fundação**; o indicador nunca garante — só desloca a odds acima do baseline.
3. **Matemático (o limite).** Num caminho sem estrutura (martingale), **E[P&L] = 0 para qualquer política**. Todo método deve declarar qual estrutura colhe e medi-la antes de construir (`ct_medir_estrutura`).

## Premissas

- **Trades vão dar errado.** Aceitar isso é premissa, não falha a eliminar.
- **A fundação sobrevive de lado arbitrário.** N momentos, cada um disparando compra E venda com o mesmo gestor — o termo direcional cancela; sobra a execução.
- **A perda é realizada, nunca escondida.** Toda posição carrega saída explícita.
- **O stop compra a assimetria.** É o que torna possível perder pouco sempre e ganhar muito às vezes.
- **O setup supera a fundação.** `Líquido(setup)` > `Líquido(piso)` no mesmo teste — senão a leitura não pagou.

## Crenças (o modo de pensar)

- **Imbalance é o primitivo.** Força = `(a − b) / (|a| + |b|)`, signed em `[-1,+1]`.
- **Atividade é o relógio.** Mede-se força onde o mercado transaciona (VWMA por volume), não no tempo uniforme.
- **Medir o fato, sem média.** Feature primária é fato de janela deslizante — extremos, pontos e ordem.
- **Força é regime.** O mesmo evento muda de significado conforme o nível (ruído/pullback/reversão).
- **Posição não é desfecho.** Estado é contexto probabilístico, não destino — armadilha é real.
- **A régua sai do fenômeno.** Âncoras e escala vêm da própria janela — o grid reage à volatilidade.
- **Gerir, não prever.** A camada adaptativa não cria EV — melhora a forma da distribuição.
- **Prevê-se regime, não preço** — e isso é SETUP, não fundação.
- **Tese só vale se sobrevive ao dado.** Robustez antes de declarar.

## A arquitetura da fundação

1. **Execução — o grupo** (`ct://libs/seed/grupo`): entradas que acumulam + saídas reduce-only em OCO.
2. **Gestão — o gestor**: a política de sobrevivência sobre o grupo. Stop curto, trailing com ativação longe, prazo. Regras adaptativas: breakeven, reescala-com-a-vol, pirâmide-fora-de-vol-alta.

## Método (as fases)

1. **Fundação** — provar o piso: o gestor sobre o grupo sobrevive de lado arbitrário (`ct_testar_sobrevivencia`).
2. **Setup** — a leitura desloca o lado acima do acaso, por cima do piso.
3. **Otimização** — tunar o sistema inteiro junto (sempre com holdout e robustez).

## Validação (o ciclo)

- **Unidade:** o teste do par — N momentos, cada um disparando compra E venda.
- **Veredito:** Σ líquido (sem taxas) ≥ 0 no agregado.
- **Diagnóstico na reprovação:**
  - Piso não passa → execução/gestão ou ativo sem estrutura.
  - Leitura não supera baseline → fica o baseline.
  - Piso OK mas setup não supera → ajustar o uso ou trocar a leitura.
- **Gatilhos:** qualquer alteração re-dispara o teste.

---

> Próximo: [O teste de sobrevivência](./02-teste-sobrevivencia.pt.md)
