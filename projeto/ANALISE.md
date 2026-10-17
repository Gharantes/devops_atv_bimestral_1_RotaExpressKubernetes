# RotaExpress: análise de 1, 2 e 3 Pods

<!--
COMO USAR ESTE MODELO
- Troque tudo que está entre [colchetes] pelos seus dados.
- Apague os comentários (como este) antes de entregar.
- Entregue junto com: resultados_modelo.csv preenchido, pasta resultados/ e pasta k8s/.
-->

**Aluno:** Guilherme Harmatiuk Arantes

**Pergunta investigada:** mantendo a mesma demanda, como a quantidade de Pods afeta o número de rotas concluídas, o tempo de resposta e as falhas? Qual é o custo em recursos de cada configuração?

**Metas da atividade:** p95 de `latencia_ok` < 800 ms · falhas < 1% · 0 iterações descartadas na medição.

---

## 1. Cenário

### Computador usado

| Item | Valor |
|---|---|
| Sistema operacional | [ex.: Windows 10 Home, PowerShell] |
| Processador | [modelo, nº de núcleos/threads] |
| Memória RAM | [ex.: 8 GB] |
| Outros programas abertos durante os testes | [ex.: só o VS Code e o navegador] |

### Recursos do cluster

| Item | Valor |
|---|---|
| Cluster | Minikube, perfil `ops-lab`, driver Docker, 1 nó |
| CPU do nó | 3 CPUs |
| Memória do nó | 3 GiB (3072 MiB) |

### Recursos por Pod

| Componente | CPU request | CPU limit | Memória request | Memória limit |
|---|---|---|---|---|
| Aplicação `rotas` (cada Pod) | 500m | 500m | 64Mi | 128Mi |
| Gerador de carga k6 (Job) | 250m | 500m | 128Mi | 256Mi |

### Parâmetros do teste

| Parâmetro | Valor |
|---|---|
| `TAXA` (pedidos iniciados por segundo) | [valor escolhido] |
| `PARADAS` | 9 |
| `DURACAO` (janela medida) | 60 s |
| Aquecimento + intervalo (fora da medição) | 10 s + 10 s |
| Repetições por configuração | 3 (rodadas A, B e C, em ordem alternada) |

### Por que escolhi essa taxa

<!-- Use o resultado do piloto (resultados/piloto.txt). Ex.: "No piloto com TAXA=6 e 1 Pod, 62% falharam e
dropped_iterations foi 0. Ou seja, a demanda supera 1 Pod, mas o k6 conseguiu gerar toda a carga. Isso torna a taxa
boa para revelar diferença entre 1, 2 e 3 Pods." -->

- Tempo de um cálculo sem limite de CPU (Docker, Passo 2): [calculo_ms] ms
- Tempo de um cálculo com meia CPU (interface, Passo 4): [tempo] ms
- Capacidade estimada de 1 Pod: ~[1000 ÷ tempo com meia CPU] pedidos/s
- Resultado do piloto: [rotas_ok, falhas %, dropped_iterations]
- [Se houve mais de um piloto, diga o que mudou e por quê.]

[Uma ou duas frases explicando a escolha.]

---

## 2. Evidências

### 2.1 Todas as execuções

<!-- Copie do resultados_modelo.csv. Sucessos/s = rotas_ok ÷ 60. -->

| Teste | Pods | rotas_ok | Sucessos/s | Falhas (%) | p95 latencia_ok (ms) | dropped_iterations | Status do Job |
|---|---|---|---|---|---|---|---|
| 3pods-a | 3 | | | | | | |
| 1pod-a | 1 | | | | | | |
| 2pods-a | 2 | | | | | | |
| 2pods-b | 2 | | | | | | |
| 3pods-b | 3 | | | | | | |
| 1pod-b | 1 | | | | | | |
| 1pod-c | 1 | | | | | | |
| 2pods-c | 2 | | | | | | |
| 3pods-c | 3 | | | | | | |

[Se algum teste foi interrompido e repetido, diga qual, o motivo e o nome do teste repetido.]

### 2.2 Medianas por configuração

<!-- Mediana = o valor do meio das 3 repetições colocadas em ordem. -->

| Pods | Sucessos/s | Falhas (%) | p95 latencia_ok (ms) | dropped_iterations | Atingiu as metas? |
|---|---|---|---|---|---|
| 1 | | | | | [sim/não, e qual meta falhou] |
| 2 | | | | | |
| 3 | | | | | |

### 2.3 Distribuição da carga entre os Pods

<!-- Use "kubectl logs -l app=rotas --prefix --tail=1" antes e depois de cada teste. Diferença = depois − antes.
Basta mostrar um teste representativo de 2 e um de 3 Pods. -->

**Teste [3pods-?]:**

| Pod | Aceitos (depois − antes) | Recusados (depois − antes) |
|---|---|---|
| [rotas-...-xxxxx] | | |
| [rotas-...-yyyyy] | | |
| [rotas-...-zzzzz] | | |

**Teste [2pods-?]:**

| Pod | Aceitos (depois − antes) | Recusados (depois − antes) |
|---|---|---|
| [rotas-...-xxxxx] | | |
| [rotas-...-yyyyy] | | |

[Uma frase: a carga ficou parecida entre os Pods ou um deles recebeu bem mais que os outros?]

### 2.4 Uso de CPU observado (`kubectl top pods`)

| Pods | CPU por Pod da aplicação (faixa observada) | CPU do k6 | Algum Pod perto de 500m? |
|---|---|---|---|
| 1 | [ex.: 480–500m] | | |
| 2 | | | |
| 3 | | | |

### 2.5 Limite do nó (Passo 8.1)

- Com 6 réplicas pedidas: [quantos Pods ficaram `Running` e quantos `Pending`]
- Evento do Pod pendente: [ex.: `0/1 nodes are available: 1 Insufficient cpu`]
- CPU requests alocados no nó (`describe node`): [ex.: 2950m (98%)]

---

## 3. Explicação

<!-- Separe sempre o que você VIU nos números do que você ACHA que é a causa. -->

### O que foi observado

- [Ex.: a mediana de sucessos/s passou de X (1 Pod) para Y (2 Pods) e Z (3 Pods).]
- [Ex.: as falhas caíram de A% para B% e C%.]
- [Ex.: o p95 ficou entre ... e ... ms em todas as configurações.]
- [Ex.: com 1 Pod, a CPU ficou perto de 500m durante todo o teste.]
- [Ex.: com 3 Pods, a carga se dividiu de forma parecida, mas ainda houve recusas em todos.]

### Hipóteses (o que provavelmente explica)

- [Ex.: com 1 Pod, a demanda (TAXA pedidos/s) supera a capacidade de ~2 pedidos/s, por isso a maior parte recebe 503.]
- [Ex.: mais Pods ajudam porque cada pedido é independente e pode ser calculado em paralelo.]
- [Ex.: mesmo com 3 Pods há falhas porque o Service escolhe o Pod sem saber se ele está ocupado. Um pedido pode cair num Pod que já está calculando e receber 503, mesmo havendo outro Pod livre (a aplicação não tem fila).]
- [Se 3 Pods ajudou pouco ou piorou: disputa de CPU no nó com o k6 e os componentes do cluster, outras tarefas no computador etc.]

### Como eu confirmaria as hipóteses

- [Ex.: repetir com TAXA menor; rodar o k6 em outro nó; olhar a distribuição por Pod.]

---

## 4. Decisão

**Configuração escolhida para esta demanda:** [N] Pods

**Custo em CPU reservada:**

| Pods | CPU reservada (requests) | Memória limite total |
|---|---|---|
| 1 | 0,5 CPU | 128 MiB |
| 2 | 1,0 CPU | 256 MiB |
| 3 | 1,5 CPU | 384 MiB |

**Justificativa:** [Ex.: "3 Pods foi a menor configuração que atingiu as metas" ou "3 Pods teve o melhor resultado,
mas ainda não atingiu a meta de falhas. Passar de 2 para 3 Pods custa mais 0,5 CPU e entregou +X sucessos/s."]

### Se nenhuma configuração atingiu as metas: próxima investigação

<!-- Preencha só se for o caso. -->

- [Ex.: testar 4 ou 5 Pods em um cluster com mais CPU, ou em mais nós.]
- [Ex.: mudar a aplicação para ter fila, ou usar um balanceador que saiba quais Pods estão ocupados.]
- [Ex.: usar um algoritmo mais rápido que a força bruta, o que reduz o tempo de cada cálculo.]
- [Ex.: configurar um HPA (autoscaling) com base na capacidade medida aqui.]

### Limite desta conclusão

Todos os Pods rodaram em **um único nó**. Este experimento mede o atendimento sob carga, mas **não** protege contra a queda desse nó nem testa recuperação de desastre. Em produção, as réplicas seriam distribuídas entre vários nós e zonas.
