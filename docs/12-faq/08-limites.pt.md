# FAQ — Limites Conhecidos

## O que NÃO é possível

| Limite | Por quê |
|---|---|
| Editar código-fonte do produto | Repo é educacional, não código-fonte |
| Aceder dados de microestrutura sem premium | Coletores são premium-gated |
| Persistir tarefas de coleta após fechar o app | Tarefas vivem em memória no servidor stdio |
| Mais de 100 séries em cache (premium) | Limite de design — LRU eviction |
| Mais de 1 série em cache (free) | Limite do tier free |
| Predict exato do preço | Por axioma: o futuro é incognoscível |

## Limites do ambiente `uv`

- Não é sandbox de segurança (não barra FS/rede de script malicioso)
- Isolamento forte (microsandbox) é frente futura
- 1ª execução pode baixar Python/libs (rede)
