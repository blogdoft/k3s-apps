# Alloy

Chart: https://artifacthub.io/packages/helm/grafana/alloy

DaemonSet (um pod por nó) coletando logs de todos os pods via
`loki.source.kubernetes` (API do Kubernetes) e enviando para o Loki. Essa
abordagem não monta `/var/log` do host nem precisa de securityContext
privilegiado — mais simples e leve que a alternativa baseada em arquivo
(estilo Promtail).

## Renderizar o template

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm template alloy grafana/alloy \
  --version 1.12.1 \
  -n observability \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repita este comando a cada upgrade de versão do chart (ajustando `--version`)
para acompanhar o diff no git.

## Configuração

O pipeline Alloy (linguagem Alloy/river) faz:
1. `discovery.kubernetes` descobre todos os pods do cluster.
2. `discovery.relabel` promove labels internos (`__meta_kubernetes_*`) para
   `namespace`, `pod`, `container`, `node_name` — sem isso os logs chegariam
   ao Loki sem nenhum label útil para busca no Grafana Explore.
3. `loki.source.kubernetes` faz o tail dos logs via API do Kubernetes.
4. `loki.write` envia para `http://loki.observability.svc.cluster.local:3100/loki/api/v1/push`.

## O que não foi configurado / requer ação manual

- **Sem secrets**: nenhuma credencial necessária (Loki sem auth).
- **RBAC**: usa o `ClusterRole`/`ClusterRoleBinding` padrão do chart — já
  cobre as permissões que `discovery.kubernetes`/`loki.source.kubernetes`
  precisam, sem overrides.
