# Análise — RotaExpress com 1, 2 e 3 Pods

Leonardo · 23/10/2026

## 1. Cenário

Todas as medições usaram a mesma demanda, de 6 pedidos por segundo, e só a quantidade de Pods mudou entre elas.

| Item | Valor |
| --- | --- |
| Computador | Windows 11 Pro, processador com 16 threads |
| Ambiente | Docker Desktop 4.88.1 (WSL 2), Minikube v1.39.0, Kubernetes v1.37.0, runtime containerd |
| Cluster | 1 nó, criado com `--cpus=3 --memory=3072` |
| Recursos por Pod da aplicação | CPU: request e limit de 500m (meia CPU). Memória: request de 64 MiB e limit de 128 MiB |
| Gerador de carga (k6, como Job) | CPU: request de 250m e limit de 500m |
| Teste | Taxa constante de 6 pedidos/s, rotas de 9 paradas, 10 s de aquecimento e 60 s de medição |
| Repetições | 3 por configuração (1, 2 e 3 Pods), em ordem alternada (rodadas A, B e C) |

A taxa foi escolhida a partir de um teste piloto e de uma estimativa de capacidade. Fora do Kubernetes, sem limite de CPU, um cálculo levou cerca de 170 ms. Com o limite de meia CPU, passou para cerca de 295 a 330 ms, então um Pod atende no máximo uns 3 pedidos por segundo.

Com 1 Pod, o piloto teve 58% de falhas, uma quantidade razoável de sucessos e nenhuma iteração descartada. A taxa de 6 pedidos/s fica entre a capacidade estimada de 1 Pod (cerca de 3/s) e a de 3 Pods (cerca de 9/s), o que permite ver diferença entre as configurações.

## 2. Evidências

Mais Pods aumentaram os sucessos e reduziram as falhas, mas nenhuma configuração atingiu a meta de menos de 1% de falhas. A tabela mostra a mediana das três repetições de cada configuração.

| Réplicas | CPU reservada | Sucessos/s | Falhas | p95 dos sucessos | Iterações descartadas | Metas atingidas |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 0,5 CPU | 2,53 | 57,89% | 369,04 ms | 0 | p95 e descartadas |
| 2 | 1,0 CPU | 3,80 | 36,84% | 366,81 ms | 0 | p95 e descartadas |
| 3 | 1,5 CPU | 4,35 | 27,70% | 363,32 ms | 0 | p95 e descartadas |

As repetições ficaram próximas entre si. Com 1 Pod, os sucessos variaram entre 2,52 e 2,55 por segundo; com 3 Pods, entre 4,18 e 4,43. Os valores de cada execução estão no arquivo `resultados_modelo.csv` e os registros completos do k6, na pasta `resultados/`.

**Distribuição entre os Pods.** Os contadores da aplicação, lidos antes e depois de cada teste, mostram que a carga foi dividida de forma parecida. Os números abaixo são de pedidos aceitos e incluem o aquecimento.

| Teste | Aceitos por Pod |
| --- | --- |
| 2pods-a | 131 / 131 |
| 2pods-b | 137 / 136 |
| 2pods-c | 135 / 132 |
| 3pods-a | 86 / 102 / 106 |
| 3pods-b | 104 / 97 / 108 |
| 3pods-c | 101 / 100 / 110 |

Em todos esses testes, a soma de aceitos e recusados nos contadores bateu exatamente com o total de pedidos registrado pelo k6.

**Uso de CPU observado** (`kubectl top pods`, durante o teste). Com 1 Pod, a leitura foi de 336m. Com 2 Pods, 301m e 323m. Com 3 Pods, entre 121m e 229m por Pod, nas duas leituras feitas. Em nenhuma leitura um Pod chegou ao limite de 500m.

## 3. Explicação

Mais réplicas ajudaram porque o cálculo pode ser feito em paralelo, mas o ganho diminuiu a cada Pod porque o Service não sabe quais Pods estão ocupados.

### O que foi observado

- Passar de 1 para 2 Pods aumentou os sucessos em cerca de 50% (de 2,53 para 3,80 por segundo). Passar de 2 para 3 aumentou só cerca de 14% (de 3,80 para 4,35).
- O p95 ficou praticamente igual nas três configurações, em torno de 365 ms. Esse valor é o tempo de um cálculo com meia CPU, porque a aplicação não tem fila: um pedido é atendido na hora ou recusado na hora.
- Com 3 Pods, cada Pod usou bem menos que o limite de 500m de CPU, e mesmo assim houve 27,7% de falhas.
- A carga foi dividida de forma equilibrada entre os Pods em todos os testes.

### Por que isso acontece (hipótese)

Cada Pod faz um cálculo por vez e responde HTTP 503 se receber outro pedido enquanto calcula. O Service escolhe o Pod de cada conexão sem saber se ele está ocupado. Assim, um pedido pode cair em um Pod que está calculando e ser recusado, mesmo havendo outro Pod livre naquele momento.

Com 1 Pod dá para ver isso com uma conta simples. Os pedidos chegam a cada 167 ms e um cálculo leva cerca de 320 ms, então o Pod consegue aceitar no máximo um pedido a cada dois, o que daria cerca de 3 sucessos por segundo. O resultado medido foi 2,53, um pouco abaixo, provavelmente porque alguns cálculos passaram de 333 ms e fizeram o Pod recusar também o pedido seguinte.

Com 3 Pods, a capacidade somada (cerca de 9 pedidos/s) já é maior que a demanda de 6/s. Por isso, o que sobra de falhas vem da escolha do Pod sem considerar a ocupação, e não da falta de CPU. Os números de CPU abaixo do limite apoiam essa hipótese, mas as leituras do `kubectl top` são amostras e servem como indício, não como prova.

## 4. Decisão

Entre as configurações testadas, eu adotaria 3 Pods, que reservam 1,5 CPU e entregaram o melhor resultado, mas nenhuma delas atende à meta de menos de 1% de falhas.

Com 3 Pods, os sucessos subiram de 2,53 para 4,35 por segundo e as falhas caíram de 57,89% para 27,70%, em relação a 1 Pod. O custo foi o triplo de CPU reservada (de 0,5 para 1,5 CPU). O último Pod adicionado, de 2 para 3, custou 0,5 CPU a mais para um ganho de apenas 14%.

Pela tendência observada, colocar mais réplicas deve trazer ganhos cada vez menores e não deve levar as falhas a menos de 1%. Por isso, a próxima investigação não seria aumentar a quantidade de Pods, e sim mudar a forma como os pedidos chegam a eles:

1. Fazer a aplicação aceitar uma pequena fila, em vez de recusar na hora quem chega enquanto ela calcula.
2. Testar um balanceamento que considere a ocupação de cada Pod, ou uma nova tentativa automática no cliente quando receber 503.
3. Repetir a mesma série de medições depois da mudança, para comparar com estes resultados.

Um autoscaler (HPA) só faria sentido depois disso. Hoje ele adicionaria Pods com base na CPU, e a CPU por Pod nem chegou ao limite enquanto as falhas aconteciam.

## 5. Observações do ambiente e limites da conclusão

- **O nó anunciou 16 CPUs, e não 3.** Com o driver Docker, o `--cpus=3` limita o consumo real do nó, mas o Kubernetes enxerga todas as CPUs do computador. Por isso, 6 réplicas couberam sem problema, e o estado `Pending` só apareceu com 32 réplicas: 2 Pods ficaram pendentes com `Insufficient cpu`, com 15950m (99%) de CPU reservada no nó. Os 30 Pods agendados estavam praticamente ociosos, o que mostra que o scheduler decide pela soma dos requests, e não pelo uso real. Isso não afetou as medições, porque cada Pod continuou limitado a 500m.
- **A readiness evitou tráfego para Pods que ainda não estavam prontos.** Todo Pod novo falhou a primeira verificação de `/readyz` por alguns segundos, e só depois entrou no Service.
- **A implantação com erro não derrubou o serviço.** Com a imagem inexistente `rotas:1.1`, o Pod novo ficou em `ImagePullBackOff`, o antigo continuou atendendo e o `rollout undo` restaurou a versão anterior.
- **Todos os Jobs do k6 terminaram como `Failed`.** Esse status vem do código de saída 99, que indica metas não atingidas. Todos os registros têm o resumo completo, então são medições válidas.
- **As leituras de CPU têm limitações.** O metrics-server atualizou os valores mais ou menos a cada 60 segundos, então algumas execuções ficaram sem leitura de CPU.
- **Um registro foi restaurado.** O arquivo `3pods-b.txt` foi sobrescrito por engano e depois recuperado de uma cópia feita antes do erro. Os valores conferem com os do terminal no momento do teste.
- **Limite da conclusão.** Os testes rodaram em um único nó, então três Pods não protegem contra a queda desse nó. Este laboratório mede atendimento sob carga e não testa recuperação de desastre; em produção, as réplicas seriam distribuídas entre nós e zonas.
