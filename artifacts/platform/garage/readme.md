# Garage

Object storage compatível com S3, em nó único (substitui o MinIO).

Chart: o Garage não publica repositório Helm — o chart vive no próprio repo
git (`script/helm/garage`). Renderizado a partir da tag `v2.4.1`
(chart `0.10.2`, app `v2.4.1`).

## Antes de renderizar / aplicar

Este repositório não guarda secrets. Crie os dois secrets manualmente no
namespace `garage` antes do primeiro sync:

```sh
kubectl create namespace garage --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic garage-rpc -n garage \
  --from-literal=rpcSecret="$(openssl rand -hex 32)"

kubectl create secret generic garage-admin -n garage \
  --from-literal=admin-token="$(openssl rand -base64 32)"
```

- `garage-rpc` (chave `rpcSecret`): segredo RPC do Garage, referenciado via
  `garage.existingRpcSecret`. Sem ele o chart geraria um valor novo a cada
  `helm template`.
- `garage-admin` (chave `admin-token`): token da admin API (`:3903`), lido pelo
  Garage (`GARAGE_ADMIN_TOKEN`) e pelo garage-webui (`API_ADMIN_KEY`).

## Renderizar o template

```sh
git clone https://git.deuxfleurs.fr/Deuxfleurs/garage.git
git -C garage checkout v2.4.1

helm template garage ./garage/script/helm/garage \
  -n garage \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repita a cada upgrade (ajustando a tag). `manifests/10-webui.yaml` é escrito à
mão (garage-webui) e não é afetado pela re-renderização.

## Layout do cluster (passo manual obrigatório)

Após o primeiro sync o pod sobe, mas não armazena nada até o layout ser
atribuído:

```sh
kubectl exec -n garage garage-0 -- /garage status        # copie o ID do nó
kubectl exec -n garage garage-0 -- /garage layout assign -z home -c 20G <ID_DO_NO>
kubectl exec -n garage garage-0 -- /garage layout apply --version 1
```

`-c` é a capacidade e deve caber no PVC de dados (20Gi). Depois crie bucket e
chave (pela UI em `garage.home.arpa` ou pelo CLI):

```sh
kubectl exec -n garage garage-0 -- /garage bucket create <bucket>
kubectl exec -n garage garage-0 -- /garage key create <nome-da-chave>
kubectl exec -n garage garage-0 -- /garage bucket allow --read --write --owner <bucket> --key <nome-da-chave>
```

## Hosts expostos (ingress)

- API S3: `garage-api.home.arpa` (região `garage`, acesso **path-style**)
- Web UI: `garage.home.arpa`

Ambos usam `ingressClassName: traefik` e o TLS default do cluster (wildcard
`wildcard-home-arpa`). Acesso vhost-style (`<bucket>.s3...`) não foi
configurado: exigiria um wildcard de dois níveis, que o certificado
`*.home.arpa` não cobre.

## O que não foi configurado / requer ação manual

- **Secrets** `garage-rpc` e `garage-admin`: manuais, ver acima.
- **Layout do cluster**: manual, ver acima.
- **Sem alta disponibilidade**: nó único, `replication_factor = 1`.
- **Autenticação do garage-webui**: sem `AUTH_USER_PASS`; a UI só é
  protegida pela rede local. Defina o hash bcrypt se quiser login.
- **Métricas Prometheus**: desabilitadas (`monitoring.metrics.enabled: false`).
- **CRD `garagenodes.deuxfleurs.fr`**: criado pelo próprio Garage em runtime
  (descoberta via Kubernetes) e não é removido ao apagar a Application.
- **Sem migração do MinIO**: os dados antigos não foram copiados.
