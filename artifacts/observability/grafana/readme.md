# Grafana

Deployment hand-escrito (não renderizado de chart), ver `manifests/deployment.yaml`.

## Antes de aplicar

Este repositório não guarda secrets. Crie o secret de credenciais admin
manualmente no namespace `observability` antes de aplicar os manifests:

```sh
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic grafana-admin -n observability \
  --from-literal=admin-user=<usuario> \
  --from-literal=admin-password=<senha-forte>
```

O nome do secret e as chaves (`admin-user` / `admin-password`) já estão
referenciados no Deployment via `secretKeyRef`.

## Datasources

Provisionados automaticamente via ConfigMap `grafana-datasources` montado em
`/etc/grafana/provisioning/datasources/`:

- **Loki** (`uid: loki`) — logs, `http://loki.observability.svc.cluster.local:3100`
- **Prometheus** (`uid: prometheus`) — métricas RED geradas pelo Tempo,
  `http://prometheus-server.observability.svc.cluster.local:80`
- **Tempo** (`uid: tempo`) — traces, `http://tempo.observability.svc.cluster.local:3200`,
  com correlação configurada: `tracesToLogsV2` (por `k8s.namespace.name`/`k8s.pod.name`
  → `namespace`/`pod`, já que o Alloy não emite um label `service_name` hoje)
  e `tracesToMetrics`/`serviceMap` apontando para o Prometheus.

Após alterar o ConfigMap, reinicie o pod
(`kubectl rollout restart deployment/grafana -n observability`) para
recarregar o provisionamento.

## Dashboards

Provisionados via ConfigMaps `grafana-dashboard-provider` (aponta a pasta
`/etc/grafana/provisioning/dashboards/json`) + `grafana-dashboards` (JSON dos
dashboards), na pasta "Observability" do Grafana:

- **Logs Overview**: volume de log por namespace, taxa de linhas
  error/exception/panic, top 10 pods por volume de log.
- **APM / Service Overview**: request rate, error rate % e latência
  p95/p99 por serviço, a partir das métricas `traces_spanmetrics_*` geradas
  pelo Tempo (ver `artifacts/observability/tempo/readme.md`). Para o mapa de
  dependências entre serviços, use a aba Node Graph do datasource Tempo em
  Explore.
- **Host & Pod Stats**: CPU/memória/disco/rede/load por nó (via
  `prometheus-node-exporter`) e CPU/memória/restarts/fase por pod (via
  cAdvisor + `kube-state-metrics`) — ver
  `artifacts/observability/prometheus/readme.md`.

## O que não foi configurado / requer ação manual

- **Credenciais admin**: precisam ser criadas manualmente como Secret
  (`grafana-admin`, namespace `observability`) antes do primeiro apply — ver acima.
