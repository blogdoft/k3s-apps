# Prometheus

Chart: https://artifacthub.io/packages/helm/prometheus-community/prometheus

Chart `prometheus-community/prometheus` puro (não `kube-prometheus-stack`),
com Alertmanager e pushgateway desabilitados. Serve como backend das
métricas RED (rate/error/duration) que o **Tempo** gera automaticamente a
partir dos traces (metrics-generator — ver
`artifacts/observability/tempo/readme.md`) **e** como fonte de métricas de
infraestrutura (host e pods) para o dashboard "Host & Pod Stats" no Grafana:

- **`prometheus-node-exporter`** (DaemonSet, um pod por nó): métricas de
  host — CPU, memória, disco, rede, load average (`node_*`).
- **`kube-state-metrics`**: estado dos objetos Kubernetes — fase do pod,
  contagem de restarts (`kube_pod_*`).
- **`kubernetes-nodes-cadvisor`** (scrape job): uso real de CPU/memória por
  pod/container, via cAdvisor do kubelet (`container_*`).

Ambos os subcharts expõem `prometheus.io/scrape: "true"` no Service, então
o job `kubernetes-service-endpoints` (descoberta por anotação) os encontra
automaticamente — sem scrape config manual por serviço.

## Remote-write receiver

`--web.enable-remote-write-receiver` habilitado (via
`server.extraArgs`) — é o que permite o Tempo empurrar as métricas de
span/service-graph para cá via `POST /api/v1/write`.

## Gotcha do chart (jobs de descoberta k8s)

Os jobs de scrape padrão do chart (`kubernetes-pods`, `kubernetes-nodes`,
etc.) vêm de um mapa `scrapeConfigs` com `enabled: true` por padrão.
Sobrescrever com uma lista ou com `{}` **não remove nada** — merge de mapa
do Helm não limpa chaves default. Cada job precisa ser desabilitado
individualmente com `enabled: false` (ver `helm-values/values.yaml`). Só o
job `prometheus` (self-scrape) fica ativo.

## Renderizar o template

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm template prometheus prometheus-community/prometheus \
  --version 29.27.1 \
  -n observability \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repita este comando a cada upgrade de versão do chart.

## Acesso interno

Sem ingress — uso apenas interno ao cluster:
- Remote-write (alvo do Tempo): `http://prometheus-server.observability.svc.cluster.local:80/api/v1/write`
- Query API (datasource Prometheus no Grafana): `http://prometheus-server.observability.svc.cluster.local:80`

## Retenção e tamanho

`retention: 15d` — mais barato que os 3 dias de logs/traces porque as
únicas séries armazenadas aqui são as métricas de baixa cardinalidade
geradas pelo Tempo (poucos serviços × poucas rotas/spans), sem
cAdvisor/kube-state-metrics/node-exporter no meio. PVC de 10Gi em
`longhorn-fast` dá folga generosa para esse volume.

## O que não foi configurado / requer ação manual

- **Sem secrets/autenticação**: nenhuma credencial a criar.
- **Sem métricas de infraestrutura** (node-exporter/kube-state-metrics):
  desabilitado deliberadamente — fora de escopo desta fase (só APM via
  Tempo). Pode ser habilitado depois como app separado se quiser dashboards
  de recursos do cluster.
- **Alta disponibilidade**: single réplica, sem federação — adequado para
  home-lab.
