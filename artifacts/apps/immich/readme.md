# Immich

Chart: https://github.com/immich-app/immich-charts (`immich/immich` 0.12.0, imagens v3.3.1)

Galeria de fotos/vídeos. Sobe o server e o machine-learning (reconhecimento
facial, busca inteligente). Postgres e Redis são **externos ao chart**:

- Postgres: externo ao cluster, precisa ter a extensão **VectorChord**
  (ou pgvecto.rs) instalada.
- Redis: o `redis-server` existente (`redis-server.redis-server.svc.cluster.local:6379`,
  com senha). O Valkey do chart está desabilitado (`valkey.enabled: false`).

## Antes de aplicar

Crie o namespace e o secret manualmente (este repo não guarda secrets):

```sh
kubectl create namespace immich --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic immich-secrets -n immich \
  --from-literal=DB_URL="postgresql://<usuario>:<senha>@<host>:5432/<database>" \
  --from-literal=REDIS_PASSWORD="<senha do redis-server>"
```

No Postgres, antes do primeiro start:

```sql
CREATE EXTENSION IF NOT EXISTS vchord CASCADE;
```

Sem o secret os pods ficam em `CreateContainerConfigError`.

## Renderizar o template

```sh
helm repo add immich https://immich-app.github.io/immich-charts
helm repo update

helm template immich immich/immich \
  --version 0.12.0 \
  -n immich \
  -f helm-values/values.yaml \
  > manifests/immich.yaml
```

O chart não cria o PVC da biblioteca: `manifests/library-pvc.yaml` é
escrito à mão (não é gerado pelo helm) e é referenciado por
`immich.persistence.library.existingClaim`.

## GPU (NVIDIA RTX 3060, nó `morgul`)

O cluster é de um nó só (`morgul`), com driver NVIDIA no host, o RuntimeClass
`nvidia` do k3s e o NVIDIA device plugin (instalado pelo `k3s-experiment`,
pasta `nvidia/`, com time-slicing de 4 réplicas). Os pods usam
`runtimeClassName: nvidia` e pedem `nvidia.com/gpu: 1`; as réplicas
compartilham os 12 GB de VRAM sem isolamento de memória.

- **Machine learning:** imagem `v3.3.1-cuda`.
- **Transcodificação:** `ffmpeg.accel: nvenc` com `accelDecode` na
  configuração do Immich (`immich.configuration`). Se o admin salvar outras
  configurações pela UI, confira se o ConfigMap não as sobrescreve.

Para validar após o deploy: `kubectl exec -n immich deploy/immich-machine-learning -- nvidia-smi`.

## Persistência

- `immich-library` (`longhorn-fast`, **100Gi**, RWO): fotos e vídeos. PVCs
  Longhorn só crescem; para aumentar, edite `library-pvc.yaml` e confira o PVC
  no cluster.
- `immich-machine-learning` (`longhorn-fast`, 10Gi): cache dos modelos de ML,
  evita rebaixá-los a cada restart.

## Hosts expostos (ingress)

`immich.home.arpa`, `ingressClassName: traefik`, cert próprio via
`cert-manager.io/cluster-issuer: home-arpa-ca` (secret `immich-tls`).

## O que não foi configurado

- **Alta disponibilidade**: uma réplica de cada componente.
- **Redis sem isolamento**: usa o DB 0 do redis compartilhado.
- **Backup**: nenhum backup do volume/banco configurado.
