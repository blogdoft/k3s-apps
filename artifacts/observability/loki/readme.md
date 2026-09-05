# Loki

Chart: https://artifacthub.io/packages/helm/grafana/loki

Modo single-binary, storage em filesystem, sem dependência do MinIO. Retenção
de logs fixada em **3 dias** via `limits_config.retention_period` +
`compactor.retention_enabled: true` — o compactor apaga os chunks/index
expirados automaticamente, sem intervenção manual.

## Rate limit de ingestão

`limits_config.ingestion_rate_mb: 16` / `ingestion_burst_size_mb: 32`
(default do chart é 4MB/s, ~6MB burst). O default se mostrou baixo demais
logo no primeiro deploy: o Alloy, ao subir, lê o backlog recente de todos os
pods do cluster de uma vez, gerando um pico que estourava o limite padrão
("ingestion rate limit exceeded").

## Renderizar o template

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm template loki grafana/loki \
  --version 7.3.0 \
  -n observability \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repita este comando a cada upgrade de versão do chart (ajustando `--version`)
para acompanhar o diff no git.

## Acesso interno

Sem ingress — uso apenas interno ao cluster. Endpoint:
`http://loki.observability.svc.cluster.local:3100` (porta `http-metrics`,
usada tanto para push/query da API HTTP quanto para métricas). Consumido pelo
Grafana (datasource) e pelo Alloy (push de logs).

## O que não foi configurado / requer ação manual

- **Sem secrets**: `auth_enabled: false`, não há autenticação nem credenciais
  a criar.
- **Métricas Prometheus**: `monitoring.*` desabilitado por padrão pelo chart
  (não alterado aqui) — fora de escopo por enquanto.
- **Alta disponibilidade**: `replication_factor: 1`, single réplica — adequado
  para uso home-lab, não tolera perda do nó/PVC.
