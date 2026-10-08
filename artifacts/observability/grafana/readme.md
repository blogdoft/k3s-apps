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
- **API Request Latency Percentiles**: latência p50, p85 e p95 por serviço e
  operação/span (normalmente a rota HTTP da API), com filtro por serviço e
  endpoint e a taxa de requisições como contexto.
- **Host & Pod Stats**: CPU/memória/disco/rede/load por nó (via
  `prometheus-node-exporter`) e CPU/memória/restarts/fase por pod (via
  cAdvisor + `kube-state-metrics`) — ver
  `artifacts/observability/prometheus/readme.md`.
- **Disk Usage**: ocupação (%) de cada PVC do cluster (`kubelet_volume_stats_*`,
  todos os namespaces, não só `observability`) e de cada filesystem de host
  (`node_filesystem_*`), com gauge + tendência e limiar visual em 80%.
- **Deployment Resources**: tabela com todos os Deployments (filtro por
  namespace) comparando o solicitado com o real: CPU/memória `request`,
  `limit`, média (Avg) e pico (Max) de uso no período selecionado no
  dashboard, e `% Over Limit` (100 × Max / Limit). Ordenável em cada coluna
  clicando no cabeçalho. Usa `kube-state-metrics` + cAdvisor; o Deployment
  é derivado do ReplicaSet dono do pod, então StatefulSets/DaemonSets não aparecem.

## Alertas

Provisionados via ConfigMap `grafana-alerting` montado em
`/etc/grafana/provisioning/alerting/`, grupo `disk-usage-alerts` na pasta
"Observability":

- **PVC usage >= 80%**: um disparo por `namespace`/`persistentvolumeclaim`
  que cruzar o limiar (`for: 10m`, evita flapping em picos curtos).
- **Host filesystem usage >= 80%**: um disparo por `instance`/`mountpoint`.

Grupo `resource-usage-alerts` (avaliação a cada 1 min, `for: 5m`): memória e
CPU de pod acima de 90% do limit do container, e CPU e memória de nó acima de 90%.

### Notificação (Telegram)

O ConfigMap também provisiona o contact point `telegram` (`contactpoints.yaml`)
e a política de notificação raiz (`policies.yaml`), que envia todos os alertas
para ele (agrupa por pasta/alertname, repete a cada 4h). Token e chat id vêm
do Secret `grafana-telegram`, injetado como env `TELEGRAM_BOT_TOKEN` /
`TELEGRAM_CHAT_ID` e referenciado via `$__env{...}`. Crie-o antes do apply:

```sh
kubectl create secret generic grafana-telegram -n observability \
  --from-literal=bot-token=<token-do-botfather> \
  --from-literal=chat-id=<chat-id>
```

Para obter o chat id: envie uma mensagem ao bot (ou adicione-o ao grupo) e
consulte `https://api.telegram.org/bot<token>/getUpdates`. Em grupos o id é
negativo. Sem o Secret, o pod do Grafana não inicia.

## O que não foi configurado / requer ação manual

- **Credenciais admin**: precisam ser criadas manualmente como Secret
  (`grafana-admin`, namespace `observability`) antes do primeiro apply — ver acima.
- **Secret do Telegram** (`grafana-telegram`): criar manualmente — ver seção
  Alertas acima.
