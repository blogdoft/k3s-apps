# external-dns

Cria e remove, na Cloudflare, os registros DNS dos Ingress públicos (`*.blogdoft.dev`), apontando-os
para o túnel (`artifacts/platform/cloudflare-tunnel`). Evita criar um CNAME à mão para cada host novo.

Chart: `external-dns/external-dns`, versão `1.23.0` (app `v0.23.0`). Roda no namespace `external-dns`.

## Como funciona

- Observa só `Ingress` (`sources: ingress`) e só a zona `blogdoft.dev` (`domainFilters`). Hosts
  `*.home.arpa` nunca são tocados.
- Só atua em Ingress com a anotação `external-dns.kubernetes.io/target` (`--annotation-filter`).
  Sem ela, o Ingress é ignorado: o CNAME continua sendo uma decisão explícita no Git.
- Registros criados por ele ganham um TXT de controle (`edns-<nome>`, dono `k3s-home`). Com
  `policy: sync` ele remove apenas o que criou; registros feitos à mão não são alterados nem apagados.
  Um registro manual com o mesmo nome **não** é sobrescrito: apague-o antes para o external-dns assumir.

## Publicar um serviço

Anote o Ingress:

```yaml
metadata:
  annotations:
    external-dns.kubernetes.io/target: 86135bb0-558c-46c8-b4a3-6ac8679cc0d2.cfargotunnel.com
    external-dns.kubernetes.io/cloudflare-proxied: "true"
```

O host também precisa estar sob `*.blogdoft.dev` para o cloudflared encaminhá-lo ao Traefik.
Em até 1 minuto o CNAME proxied aparece na Cloudflare.

## Secret (criado manualmente, fora do Git)

Crie um API token **exclusivo** para este app em https://dash.cloudflare.com/profile/api-tokens
(Create Token → Custom token), com as permissões e o escopo mínimos:

| Permissão        | Nível | Acesso |
| ---------------- | ----- | ------ |
| Zone → DNS       | Zone  | Edit   |
| Zone → Zone      | Zone  | Read   |

Em *Zone Resources*, use `Include → Specific zone → blogdoft.dev`. Não reutilize o token do cert-manager.
O `cloudflared` CLI não cria API tokens (ele só gerencia túneis), então a criação é pelo painel ou
pela API da Cloudflare.

O Secret `external-dns-cloudflare` (chave `api-token`) é criado pelo `install.sh` do repositório de
bootstrap (`k3s-experiment`), a partir do `.env`:

```sh
# em k3s-experiment/.env
export CLOUDFLARE_EXTERNAL_DNS_TOKEN="<token>"

# em k3s-experiment/
./cloudflare-tunnel/install.sh
```

O script cria o namespace `external-dns` (se não existir) e aplica o Secret a partir do template
`cloudflare-tunnel/external-dns-cloudflare-secret.yaml.template` com `envsubst`. Ele também aplica o
Secret de credenciais do túnel e pode ser reexecutado sem efeitos colaterais.

## Renderizar o template

```sh
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

helm template external-dns external-dns/external-dns \
  --version 1.23.0 \
  -n external-dns \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

## Verificar

```sh
kubectl get pods -n external-dns
kubectl logs -n external-dns deploy/external-dns
dig +short @1.1.1.1 immich.blogdoft.dev
```

## Cuidados

- O token edita o DNS de `blogdoft.dev`: mantenha o escopo restrito a essa zona.
- Trocar o token e reexecutar o `install.sh` não reinicia o pod (o Deployment não usa o Reloader): rode
  `kubectl rollout restart deploy/external-dns -n external-dns`.
