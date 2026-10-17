# Kafka

Broker único em modo **KRaft** (sem ZooKeeper), imagem `apache/kafka:3.9.0`,
manifestos escritos à mão (sem Helm). O mesmo processo é broker e controller.

- Namespace: `kafka`; StatefulSet `kafka` (1 réplica), PVC de 5Gi em `longhorn-fast`.
- Recursos mínimos: heap 384m, request 100m/512Mi, limit 500m/640Mi.
- Retenção: 72h ou 1GiB por partição; 1 partição por tópico por padrão.
- Todos os fatores de replicação / min ISR = 1 (obrigatório com 1 broker).

## Acesso

| Origem | Endereço |
| --- | --- |
| Dentro do cluster | `kafka.kafka.svc.cluster.local:9092` |
| LAN / fora do cluster | `kafka-bootstrap.home.arpa:9092` |

O listener externo é o Service `kafka-external` (`LoadBalancer`, ServiceLB do K3s,
porta 9092 → 9094 do pod). `kafka-bootstrap.home.arpa` precisa resolver para o IP
de um nó do cluster, e o `advertised.listeners` externo usa esse mesmo nome. É o
endereço que o `kafka-ui` já usa.

## Métricas (JMX)

O broker expõe JMX na porta 9997 (registry e RMI na mesma porta), usado pelo `kafka-ui`
via `kafka-0.kafka-headless.kafka.svc.cluster.local:9997`. Fica só dentro do cluster: não
está no Service `kafka-external`, pois está sem autenticação/TLS.

## Não configurado de propósito

- Sem TLS/SASL (PLAINTEXT): restrinja o acesso à LAN.
- Sem alta disponibilidade: reiniciar o pod indisponibiliza o Kafka por alguns segundos.
- `CLUSTER_ID` fixo no manifesto; trocá-lo com dados no PVC impede o broker de subir.
