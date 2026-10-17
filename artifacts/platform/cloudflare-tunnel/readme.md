# cloudflare-tunnel

Expõe serviços do cluster na internet por um Cloudflare Tunnel, sem abrir portas no roteador. O
`cloudflared` roda dentro do cluster e abre conexões de saída para a Cloudflare.

Manifests escritos à mão (não há chart/Helm). Imagem: `cloudflare/cloudflared:2026.10.1`.

## Como funciona

- `manifests/00-deployment.yaml`: ServiceAccount `cloudflared` (sem token de API montado) e Deployment com
  2 réplicas, rodando como não-root, com filesystem somente leitura e sem capabilities.
- `manifests/01-configmap.yaml`: `config.yaml` do túnel. Regras de ingress:
  1. `*.blogdoft.dev` → `http://traefik.kube-system.svc.cluster.local:80`. O Host original é preservado e
     o Traefik escolhe o Ingress correspondente.
  2. Qualquer outro hostname → `http_status:404` (regra final obrigatória do cloudflared).
- Para publicar um serviço, crie um Ingress `ingressClassName: traefik` com host `<app>.blogdoft.dev`
  (exemplo: `immich-server-public`) e um CNAME no DNS da Cloudflare apontando para
  `<tunnel-id>.cfargotunnel.com` (proxied). **Todo Ingress com host `*.blogdoft.dev` fica público.**

## Secret (criado manualmente, fora do Git)

O Secret `cloudflared-credentials` precisa existir antes do primeiro sync, com a chave `credentials.json`
gerada na criação do túnel (`cloudflared tunnel create <nome>`):

```sh
kubectl create secret generic cloudflared-credentials \
  -n cloudflare-tunnel \
  --from-file=credentials.json=$HOME/.cloudflared/<tunnel-id>.json
```

O ID do túnel (`86135bb0-558c-46c8-b4a3-6ac8679cc0d2`) está no ConfigMap e não é segredo.

## Atualizar a versão

Troque a tag da imagem em `manifests/00-deployment.yaml` (releases em
https://github.com/cloudflare/cloudflared/releases) e faça commit; o ArgoCD aplica.

## Verificar

```sh
kubectl get pods -n cloudflare-tunnel
kubectl logs -n cloudflare-tunnel deploy/cloudflared | grep -i "registered tunnel connection"
curl -I https://immich.blogdoft.dev
```

## Cuidados

- O Deployment tem `reloader.stakater.com/auto: "true"`: alterar o ConfigMap ou o Secret reinicia os pods
  automaticamente (rolling update, sem derrubar o túnel).
- O tráfego entra em HTTP na porta 80 do Traefik; o TLS público termina na Cloudflare.
- `manifests/02-networkpolicy.yaml` bloqueia todo ingress e limita o egress a: DNS (CoreDNS), Traefik
  (porta `8000`, o targetPort) e a internet pública na `7844` (borda da Cloudflare). Redes privadas
  (`10/8`, `172.16/12`, `192.168/16`) ficam bloqueadas. Se o cloudflared precisar de outro destino
  (ex.: `443` para a API da Cloudflare), a política precisa ser ajustada, ou o túnel cai.
- `manifests/03-pdb.yaml` (`minAvailable: 1`) impede remover as duas réplicas por operações voluntárias.
  Em cluster de um nó só, `kubectl drain` fica travado nesse PDB: apague-o temporariamente nesse caso.
