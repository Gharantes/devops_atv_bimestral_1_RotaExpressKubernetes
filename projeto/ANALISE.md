# RotaExpress: análise de 1, 2 e 3 Pods

**Aluno:** Guilherme Harmatiuk Arantes

**Pergunta investigada:** mantendo a mesma demanda, como a quantidade de Pods afeta o número de rotas concluídas, o tempo de resposta e as falhas? Qual é o custo em recursos de cada configuração?

**Metas da atividade:** p95 de `latencia_ok` < 800 ms · falhas < 1% · 0 iterações descartadas na medição.

---

## 1. Cenário

### Computador usado

| Item | Valor |
|---|---|
| Sistema operacional | Windows 10 Home N (10.0.19045), PowerShell 5.1, Docker Desktop com WSL2 |
| Processador | Intel Core i5-10210U @ 1,60 GHz, 4 núcleos / 8 threads |
| Memória RAM | 7,8 GB |
| Outros programas abertos durante os testes | VS Code, Docker Desktop, Firefox e Explorador de Arquivos do Windows |

### Recursos do cluster

| Item | Valor |
|---|---|
| Cluster | Minikube, perfil `ops-lab`, driver Docker, 1 nó |
| CPU do nó | 3 CPUs (limite do contêiner no Docker; o kubelet anuncia 8, ver 2.5) |
| Memória do nó | 3 GiB (3072 MiB) |

### Recursos por Pod

| Componente | CPU request | CPU limit | Memória request | Memória limit |
|---|---|---|---|---|
| Aplicação `rotas` (cada Pod) | 500m | 500m | 64Mi | 128Mi |
| Gerador de carga k6 (Job) | 250m | 500m | 128Mi | 256Mi |

### Parâmetros do teste

| Parâmetro | Valor |
|---|---|
| `TAXA` (pedidos iniciados por segundo) | 6 |
| `PARADAS` | 9 |
| `DURACAO` (janela medida) | 60 s |
| Aquecimento + intervalo (fora da medição) | 10 s + 10 s |
| Repetições por configuração | 3 (rodadas A, B e C, em ordem alternada) |

### Por que escolhi essa taxa

- Tempo de um cálculo sem limite de CPU (Docker, Passo 2): 286,1 ms
- Tempo de um cálculo com meia CPU (interface, Passo 4): 606,7 ms
- Capacidade estimada de 1 Pod: ~1000 ÷ 606,7 ≈ 1,6 pedidos/s (2 Pods ≈ 3,3/s; 3 Pods ≈ 4,9/s)
- Resultado do piloto (1 Pod, TAXA=6): `rotas_ok` = 90 (1,50/s), falhas = 75,06% (271 de 361), p95 = 627,82 ms, `dropped_iterations` = 0
- Houve um único piloto.

No piloto, a demanda superou bem a capacidade de 1 Pod (75% de falhas), mas ainda houve uma quantidade razoável de sucessos e `dropped_iterations` foi 0, ou seja, o k6 conseguiu gerar toda a carga pedida. Pelo critério do guia, isso é um bom ponto de partida. Como 6 pedidos/s também fica acima da capacidade estimada de 3 Pods (~4,9/s), a mesma taxa coloca pressão nas três configurações e deixa visível a diferença entre elas. Por isso mantive `TAXA=6` em toda a série.

---

## 2. Evidências

### 2.1 Todas as execuções

| Teste | Pods | rotas_ok | Sucessos/s | Falhas (%) | p95 latencia_ok (ms) | dropped_iterations | Status do Job |
|---|---|---|---|---|---|---|---|
| 3pods-a | 3 | 176 | 2,93 | 51,24 | 720,42 | 0 | Failed |
| 1pod-a | 1 | 90 | 1,50 | 75,06 | 641,24 | 0 | Failed |
| 2pods-a | 2 | 148 | 2,47 | 59,00 | 647,49 | 0 | Failed |
| 2pods-b | 2 | 148 | 2,47 | 59,00 | 642,96 | 0 | Failed |
| 3pods-b | 3 | 176 | 2,93 | 51,24 | 706,93 | 0 | Failed |
| 1pod-b | 1 | 90 | 1,50 | 75,06 | 657,80 | 0 | Failed |
| 1pod-c | 1 | 90 | 1,50 | 75,00 | 635,59 | 0 | Failed |
| 2pods-c | 2 | 150 | 2,50 | 58,44 | 638,08 | 0 | Failed |
| 3pods-c | 3 | 180 | 3,00 | 50,13 | 670,08 | 0 | Failed |

Nenhum teste foi interrompido ou repetido. Todos os Jobs terminaram como `Failed` porque a meta de falhas (< 1%) não foi atingida (k6 sai com código 99). O resumo completo de cada teste está em `resultados/`.

### 2.2 Medianas por configuração

| Pods | Sucessos/s | Falhas (%) | p95 latencia_ok (ms) | dropped_iterations | Atingiu as metas? |
|---|---|---|---|---|---|
| 1 | 1,50 | 75,06 | 641,24 | 0 | Não: falhas ≥ 1% (p95 e dropped OK) |
| 2 | 2,47 | 59,00 | 642,96 | 0 | Não: falhas ≥ 1% (p95 e dropped OK) |
| 3 | 2,93 | 51,24 | 706,93 | 0 | Não: falhas ≥ 1% (p95 e dropped OK) |

As três repetições de cada configuração ficaram muito próximas (90/90/90, 148/148/150 e 176/176/180 rotas), então as medianas são representativas.

### 2.3 Distribuição da carga entre os Pods

Os totais incluem o aquecimento (421 pedidos por teste).

**Teste 3pods-c:**

| Pod | Aceitos (depois − antes) | Recusados (depois − antes) |
|---|---|---|
| rotas-fc7784c5c-89kjc | 71 | 75 |
| rotas-fc7784c5c-hnp8s | 69 | 71 |
| rotas-fc7784c5c-lp25l | 71 | 65 |

**Teste 2pods-b:**

| Pod | Aceitos (depois − antes) | Recusados (depois − antes) |
|---|---|---|
| rotas-fc7784c5c-knjpd | 86 | 130 |
| rotas-fc7784c5c-lp25l | 84 | 121 |

A carga ficou praticamente igual entre os Pods em todos os testes (diferença de no máximo 4 aceitos entre um Pod e outro). Nenhum Pod recebeu bem mais que os outros, e todos recusaram uma parte parecida.

### 2.4 Uso de CPU observado (`kubectl top pods`)

| Pods | CPU por Pod da aplicação (faixa observada) | CPU do k6 | Algum Pod perto de 500m? |
|---|---|---|---|
| 1 | 309–376m (pico) | não medido | Não, ficou entre 60% e 75% do limite |
| 2 | 318–364m (pico) | não medido | Não |
| 3 | sem leitura confiável (ver abaixo) | não medido | — |

Limitações da medição de CPU: o `metrics-server` atualiza o valor a cada ~15–60 s, então as primeiras leituras de cada teste ainda mostravam o período ocioso anterior. Nos testes 3pods-b e 3pods-c, a leitura não chegou a atualizar dentro da janela de 60 s (78–172m, valores do período ocioso). No 3pods-a, o `top` foi lido logo depois do fim do teste (151–173m). A CPU do k6 não foi registrada.

### 2.5 Limite do nó (Passo 8.1)

- Com 6 réplicas pedidas: **6 `Running`, 0 `Pending`**. O contêiner do Minikube está limitado a 3 CPUs pelo Docker (`NanoCpus=3000000000`), mas o kubelet anuncia as **8 CPUs lógicas** do computador (`Allocatable cpu: 8`). O scheduler decide pelo valor anunciado, então os 6 Pods couberam (requests alocados: 3950m, 49%).
- Para ver o limite, pedi 16 réplicas: **14 `Running` e 2 `Pending`**.
- Evento do Pod pendente: `0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.`
- CPU requests alocados no nó (`describe node`) com 16 réplicas: 7950m (99%).

Registro completo em `resultados/passo8-limite-no.txt`.

---

## 3. Explicação

### O que foi observado

- A mediana de sucessos/s passou de **1,50** (1 Pod) para **2,47** (2 Pods) e **2,93** (3 Pods).
- O ganho diminuiu a cada Pod: de 1 para 2 Pods foram +0,97 sucessos/s; de 2 para 3, só +0,46. Por CPU reservada, foram 3,0 sucessos/s por CPU com 1 Pod, 2,5 com 2 e 2,0 com 3.
- As falhas caíram de **75,06%** para **59,00%** e **51,24%**, mas ficaram muito longe da meta de 1% em todas as configurações.
- O p95 ficou entre 636 e 658 ms com 1 e 2 Pods e subiu para 670–720 ms com 3 Pods. Sempre ficou abaixo de 800 ms.
- `dropped_iterations` foi 0 em todos os testes: o k6 sempre gerou a carga pedida.
- Com 1 e 2 Pods, a CPU de cada Pod ficou em ~310–375m no pico, abaixo do limite de 500m.
- A carga se dividiu de forma quase igual entre os Pods, e mesmo assim todos os Pods recusaram pedidos.
- O tempo de cálculo registrado pelos Pods durante a carga ficou entre ~550 e ~700 ms, próximo dos 606,7 ms medidos na interface.

### Hipóteses (o que provavelmente explica)

- **Com 1 Pod, a demanda supera a capacidade.** Um cálculo leva ~0,6 s com meia CPU, então 1 Pod atende no máximo ~1,6 pedidos/s. Com 6 pedidos/s chegando, a maior parte recebe 503. O valor medido (1,50/s) está bem perto dessa estimativa.
- **Mais Pods ajudam porque cada pedido é independente** e pode ser calculado em paralelo em outro Pod. Por isso os sucessos subiram com 2 e 3 Pods.
- **O ganho é bem menor que o teórico** (3 Pods ≈ 4,9/s; medido: 2,93/s) **porque a aplicação não tem fila.** Cada Pod aceita um cálculo por vez, e o Service escolhe o Pod sem saber se ele está ocupado. Um pedido pode cair num Pod que já está calculando e receber 503, mesmo com outro Pod livre. Um modelo simples de "perda sem fila" (cada Pod recebe 6 ÷ N pedidos/s e fica ocupado ~0,6 s; fração recusada ≈ ρ ÷ (1 + ρ), com ρ = 3,6 ÷ N) prevê ~78%, ~64% e ~55% de falhas para 1, 2 e 3 Pods. Os valores medidos (75%, 59%, 51%) seguem a mesma tendência. Pelo mesmo modelo, chegar a menos de 1% de falhas exigiria centenas de Pods. Ou seja, o gargalo é o desenho "um pedido por vez, sem fila", e não a quantidade de Pods.
- **A CPU não foi o gargalo direto com 1 e 2 Pods:** os Pods ficaram abaixo de 500m porque passam parte do tempo ociosos entre um pedido aceito e o próximo (os recusados quase não gastam CPU).
- **O p95 maior com 3 Pods** provavelmente vem da disputa de CPU no nó: 3 Pods × 0,5 CPU + k6 (até 0,5 CPU) + componentes do cluster, tudo dentro dos 3 CPUs reais do contêiner do Minikube, e ainda VS Code, Docker Desktop e Firefox no mesmo computador. Cálculos simultâneos ficam um pouco mais lentos.
- **O scheduler não enxerga o limite real de 3 CPUs** (ele vê 8). Ele aceitaria ~15 Pods, mas todos dividiriam os mesmos 3 CPUs reais. Acima de ~5 Pods, mais réplicas não acrescentariam capacidade real neste computador.

### Como eu confirmaria as hipóteses

- Repetir a série com `TAXA` menor (por exemplo 2 e 4) e comparar as falhas com o modelo de perda sem fila.
- Medir a CPU com `kubectl top pods` em intervalos curtos durante toda a janela (incluindo o k6), ou acompanhar pelo `docker stats ops-lab`, para confirmar a disputa de CPU com 3 Pods.
- Rodar 3 Pods com o k6 em outro nó, ou com menos programas abertos no computador, para ver se o p95 volta para ~640 ms.
- Testar uma versão da aplicação com fila (ou com mais de um cálculo por Pod) para ver se as falhas caem mesmo com poucos Pods.

---

## 4. Decisão

**Configuração escolhida para esta demanda:** 3 Pods

**Custo em CPU reservada:**

| Pods | CPU reservada (requests) | Memória limite total |
|---|---|---|
| 1 | 0,5 CPU | 128 MiB |
| 2 | 1,0 CPU | 256 MiB |
| 3 | 1,5 CPU | 384 MiB |

**Justificativa:** nenhuma configuração atingiu as metas: a meta de falhas (< 1%) falhou em todas, enquanto p95 e `dropped_iterations` foram atendidos em todas. Entre as opções testadas, 3 Pods teve o melhor resultado: 2,93 sucessos/s e 51% de falhas, com p95 de 707 ms, ainda dentro da meta. Passar de 2 para 3 Pods custa mais 0,5 CPU e entregou +0,46 sucessos/s (+19%). É um ganho menor que o de 1 para 2 (+0,97/s pelo mesmo custo), mas ainda positivo, e o nó tem CPU sobrando para isso. Se o custo de CPU fosse a prioridade, 2 Pods seria a opção mais eficiente por CPU. Mas com metade dos pedidos ainda falhando, servir mais clientes pesa mais. Também fica claro que **só aumentar o número de Pods não resolve**: o ganho cai a cada Pod e não chega perto de 1% de falhas.

### Se nenhuma configuração atingiu as metas: próxima investigação

- Mudar a aplicação para ter uma fila (ou aceitar alguns cálculos em paralelo por Pod), para que um pedido espere alguns milissegundos em vez de receber 503 na hora. Esta é a mudança com maior efeito esperado.
- Usar um algoritmo mais rápido que a força bruta (9! ordens), o que reduz o tempo de cada cálculo e aumenta a capacidade de cada Pod.
- Usar um balanceamento que saiba quais Pods estão ocupados (por exemplo, "menos conexões"), em vez da escolha aleatória do Service.
- Só depois disso, testar 4 ou 5 Pods em um cluster com mais CPU real ou em mais nós, e configurar um HPA (autoscaling) com base na capacidade medida aqui.

### Limite desta conclusão

Todos os Pods rodaram em **um único nó**. Este experimento mede o atendimento sob carga, mas **não** protege contra a queda desse nó nem testa recuperação de desastre. Em produção, as réplicas seriam distribuídas entre vários nós e zonas. Além disso, o computador estava com outros programas abertos (VS Code, Docker Desktop, Firefox e Explorador de Arquivos), que disputam a mesma CPU, e as leituras de CPU dos testes com 3 Pods não foram confiáveis.
