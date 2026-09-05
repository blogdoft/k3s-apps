# MinIO

Chart: https://artifacthub.io/packages/helm/bitnami/minio

## Antes de renderizar

Este repositório não guarda secrets. Crie o secret de credenciais root
manualmente no namespace `minio` antes de aplicar os manifests:

```sh
kubectl create namespace minio --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic minio-credentials -n minio \
  --from-literal=root-user=<usuario> \
  --from-literal=root-password=<senha-forte>
```

O nome do secret e as chaves usadas (`root-user` / `root-password`) já estão
configurados em `helm-values/values.yaml` via `auth.existingSecret`.

## Renderizar o template

```sh
helm template minio oci://registry-1.docker.io/bitnamicharts/minio \
  --version 17.0.21 \
  -n minio \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repita este comando a cada upgrade de versão do chart (ajustando `--version`)
para acompanhar o diff no git.

## Hosts expostos (ingress)

- Console (UI): `minio.home.arpa`
- API S3: `minio-api.home.arpa`

Ambos usam `ingressClassName: traefik` e o TLS default do cluster
(certificado wildcard `wildcard-home-arpa` via TLSStore do Traefik), sem
precisar referenciar nenhum secret de TLS nos manifests deste app.

> Nota: a API foi exposta em `minio-api.home.arpa` (não `s3.minio.home.arpa`)
> porque o certificado wildcard cobre apenas `*.home.arpa` (um nível de
> subdomínio). Um host de dois níveis não seria coberto pelo certificado.

## O que não foi configurado / requer ação manual

- **Credenciais root**: precisam ser criadas manualmente como Secret
  (`minio-credentials`, namespace `minio`) antes do primeiro apply — ver acima.
- **Métricas Prometheus**: desabilitadas por padrão (`metrics.enabled: false`).
