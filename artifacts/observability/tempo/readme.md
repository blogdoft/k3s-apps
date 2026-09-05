# Tempo

Chart: https://artifacthub.io/packages/helm/grafana/tempo

Modo single-binary, storage em filesystem (mesmo princípio do Loki, sem
dependência de object storage externo). Retenção de traces fixada em
**3 dias** via `tempo.retention: 72h`.

> Nota: o chart `grafana/tempo` está marcado como `deprecated` (o Grafana
> está migrando esse chart para outro repositório). Ele ainda renderiza e
> funciona corretamente hoje — usado por ora pelo mesmo motivo do Loki/Alloy
> (mesmo repo Helm, mesmo ciclo de vida). Uma migração de chart pode ser
> necessária no futuro; ao fazê-la, revalidar os values abaixo com
> `helm template` antes de aplicar.

## Metrics-generator (a base do APM)

Habilitado (`metricsGenerator.enabled: true`) com os processors
`service-graphs` e `span-metrics` — **sem esse `overrides.defaults` explícito
a mais, o generator fica ligado mas sem nenhum processor ativo** (gotcha do
chart, confirmado renderizando). É isso que transforma traces brutos em
métricas RED (rate/error/duration) por serviço, enviadas via remote_write
para `http://prometheus-server.observability.svc.cluster.local:80/api/v1/write`.

## Renderizar o template

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm template tempo grafana/tempo \
  --version 1.24.4 \
  -n observability \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repita este comando a cada upgrade de versão do chart.

## Acesso interno

Sem ingress — uso apenas interno ao cluster:
- OTLP (só o OTel Collector deve falar aqui): `tempo.observability.svc.cluster.local:4317` (grpc) / `:4318` (http)
- Query API (consumida pelo datasource Tempo no Grafana): `http://tempo.observability.svc.cluster.local:3200`

Não há `NetworkPolicy` no cluster restringindo isso — "só o Collector fala
com o Tempo" é uma convenção documentada, não uma garantia técnica (mesmo
espírito do `auth_enabled: false` do Loki).

## O que não foi configurado / requer ação manual

- **Sem secrets/autenticação**: nenhuma credencial a criar.
- **Alta disponibilidade**: single binary, sem réplica — adequado para
  home-lab, não tolera perda do nó/PVC.
