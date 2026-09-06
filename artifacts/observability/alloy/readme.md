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
2. `discovery.relabel` primeiro **descarta** (`action = "drop"`) os pods dos
   namespaces de infraestrutura/plataforma que não devem ter logs coletados
   — ver lista abaixo — e só então promove labels internos
   (`__meta_kubernetes_*`) para `namespace`, `pod`, `container`, `node_name`
   nos pods restantes.
3. `loki.source.kubernetes` faz o tail dos logs via API do Kubernetes.
4. `loki.write` envia para `http://loki.observability.svc.cluster.local:3100/loki/api/v1/push`.

### Namespaces excluídos da coleta

`argocd`, `longhorn-system`, `minio`, `cattle-*`/`fleet-*` (Rancher —
inclui `cattle-system`, `cattle-capi-system`, `cattle-fleet-local-system`,
`cattle-fleet-system`, `cattle-turtles-system`, `fleet-default`,
`fleet-local`), `redis-server`, `flagr`, `kafka-ui`, `keycloak`, `openbao`,
`observability` (o próprio stack de observabilidade) e `kube-system`.

Para adicionar/remover um namespace da exclusão, edite o regex do primeiro
`rule { action = "drop" }` em `helm-values/values.yaml` e re-renderize.

## O que não foi configurado / requer ação manual

- **Sem secrets**: nenhuma credencial necessária (Loki sem auth).
- **RBAC**: usa o `ClusterRole`/`ClusterRoleBinding` padrão do chart — já
  cobre as permissões que `discovery.kubernetes`/`loki.source.kubernetes`
  precisam, sem overrides.
