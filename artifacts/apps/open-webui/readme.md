# Open WebUI

Chart: https://artifacthub.io/packages/helm/open-webui/open-webui

Interface web para modelos servidos pelo Ollama (rodando fora do cluster,
em `192.168.1.212:11434`) e, opcionalmente, pela API da OpenAI. `ollama.enabled: false`
porque o Ollama não é gerenciado por este chart — só é referenciado via
`ollamaUrls`.

## Antes de aplicar

Este repositório não guarda secrets. Crie o secret de conexão com o
Postgres manualmente no namespace `open-webui` antes do primeiro apply:

```sh
kubectl create namespace open-webui --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic open-webui-database -n open-webui \
  --from-literal=DATABASE_URL="postgresql://<usuario>:<senha>@<host>:5432/<database>"
```

A chave (`DATABASE_URL`) já está referenciada no Deployment via
`extraEnvVars`/`secretKeyRef` em `helm-values/values.yaml`. Sem esse secret
o pod não sobe (`CreateContainerConfigError`).

## Renderizar o template

```sh
helm repo add open-webui https://helm.openwebui.com/
helm repo update

helm template open-webui open-webui/open-webui \
  --version 16.0.0 \
  -n open-webui \
  -f helm-values/values.yaml \
  > manifests/open-webui.yaml
```

Repita este comando a cada mudança em `helm-values/values.yaml` ou a cada
upgrade de versão do chart (ajustando `--version`) para acompanhar o diff no
git — o ArgoCD sincroniza `manifests/` diretamente, não o `helm-values/`
(esse arquivo existe só para permitir re-renderizar).

## Persistência

PVC `longhorn-fast` montado em `/app/backend/data` — guarda banco local
(não usado quando `DATABASE_URL` aponta pro Postgres, ver acima), banco
vetorial do RAG (`vector_db/chroma.sqlite3`) e, principalmente, o **cache de
modelos de ML baixados sob demanda**:

- `cache/embedding/models` — modelo de embedding usado pelo RAG (padrão da
  aplicação: `sentence-transformers/all-MiniLM-L6-v2`, ~90MB, mas o chart
  baixa todas as variantes do repositório no HuggingFace — PyTorch, ONNX,
  OpenVINO — inflando o cache para ~700MB mesmo sem uso das variantes
  extras).
- `cache/whisper/models` — modelo de transcrição de áudio (~140MB).

Tamanho atual: **1.5Gi** (aumentado de 1Gi — o valor original enchia com
apenas os dois modelos acima, e downloads que falhavam por falta de espaço
deixavam blobs `*.incomplete` órfãos, piorando o problema). PVCs do
Longhorn não encolhem, só crescem: para aumentar de novo, edite
`persistence.size` aqui, re-renderize, e depois expanda o PVC já existente
no cluster (o `kubectl edit pvc`/`storage:` do chart não redimensiona um PVC
existente sozinho).

Se o volume encher de novo, verifique lixo acumulado antes de aumentar o
PVC:

```sh
kubectl exec -n open-webui open-webui-0 -- find /app/backend/data/cache -name "*.incomplete"
```

## Hosts expostos (ingress)

`open-webui.home.arpa`, `ingressClassName: traefik`, `tls: false` no
values — usa o certificado wildcard default do cluster via `TLSStore` do
Traefik, sem precisar referenciar secret de TLS aqui.

## O que não foi configurado / requer ação manual

- **Credenciais do Postgres**: precisam ser criadas manualmente como Secret
  (`open-webui-database`, namespace `open-webui`) antes do primeiro apply —
  ver acima.
- **Alta disponibilidade**: `replicaCount: 1`, sem réplica — adequado para
  home-lab, não tolera perda do nó/PVC.
- **Websocket manager/Redis**: `websocket.redis.enabled: false` — não
  necessário com uma única réplica; precisa ser habilitado se
  `replicaCount` for aumentado.
