# Prometheus

Chart: https://artifacthub.io/packages/helm/prometheus-community/prometheus

Instalação enxuta — chart `prometheus-community/prometheus` puro (não
`kube-prometheus-stack`), com Alertmanager, node-exporter, kube-state-metrics
e pushgateway todos desabilitados. Neste stack, o Prometheus serve
principalmente como backend das métricas RED (rate/error/duration) que o
**Tempo** gera automaticamente a partir dos traces (metrics-generator —
ver `artifacts/observability/tempo/readme.md`), não como um monitor de
infraestrutura do cluster.

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
