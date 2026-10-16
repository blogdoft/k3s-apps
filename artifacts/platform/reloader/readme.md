# Reloader

Reinicia Deployments/StatefulSets/DaemonSets quando um ConfigMap ou Secret que eles usam muda.
Sem ele, variáveis de ambiente (`envFrom`) e arquivos montados só são relidos na próxima partida do pod.

Chart: `stakater/reloader`, versão `2.2.18` (app `v1.4.22`). Roda no namespace `kube-system` e observa
todos os namespaces (`watchGlobally: true`).

## Habilitar em um workload

Anote o Deployment (não o pod):

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

`auto` reinicia quando **qualquer** ConfigMap ou Secret referenciado pelo Deployment muda (via `env`,
`envFrom` ou volume). Para observar só recursos específicos use
`configmap.reloader.stakater.com/reload: "nome"` ou `secret.reloader.stakater.com/reload: "nome"`.

## Estratégia de reinício

`reloadStrategy: annotations`: o Reloader escreve uma anotação no template do pod
(`reloader.stakater.com/last-reloaded-from`). A estratégia padrão (`env-vars`) injetaria uma variável de
ambiente no Deployment, o que o ArgoCD (com `selfHeal`) trataria como drift e reverteria.

## Renderizar o template

```sh
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update

helm template reloader stakater/reloader \
  --version 2.2.18 \
  -n kube-system \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

## Verificar

```sh
kubectl get deploy -n kube-system reloader-reloader
kubectl logs -n kube-system deploy/reloader-reloader --tail=20
```

## Cuidados

- Ele reinicia pods a cada mudança de ConfigMap/Secret anotado, inclusive as feitas por engano.
  Mudanças em Secrets de banco, por exemplo, derrubam e recriam os pods na hora.
- Está sem alta disponibilidade (`enableHA: false`): se o pod do Reloader cair, as mudanças feitas
  nesse intervalo só são percebidas no próximo evento (`reloadOnCreate` está desligado).
