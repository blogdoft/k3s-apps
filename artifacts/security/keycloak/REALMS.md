# Bootstrap de realms/clients no Keycloak (GitOps)

Este setup faz o Keycloak iniciar com `--import-realm` e carregar os arquivos JSON presentes em `/opt/keycloak/data/import`.
No Kubernetes, isso é alimentado pelo ConfigMap `keycloak-realm-import`.

## Como exportar do Keycloak atual

1) Entre no Pod do Keycloak:

`kubectl -n keycloak exec -it statefulset/keycloak -- /bin/bash`

2) Faça o export para um diretório temporário:

`/opt/keycloak/bin/kc.sh export --dir /tmp/keycloak-export --users same_file`

3) Liste os arquivos gerados:

`ls -lah /tmp/keycloak-export`

Normalmente você terá um ou mais arquivos `*-realm.json`.

## Como versionar no repo

Edite o ConfigMap em [artifacts/security/keycloak/manifests/keycloak-realm-import-configmap.yaml](artifacts/security/keycloak/manifests/keycloak-realm-import-configmap.yaml) e adicione cada export como uma chave nova em `data:`.

Exemplo:

- `myrealm-realm.json: |`
- (cole o JSON exportado, mantendo a indentação)

Depois o ArgoCD aplicará o ConfigMap e o Keycloak, em startups futuros, tentará importar os realms.

## Importante (comportamento)

- O import no startup geralmente **não sobrescreve** realms que já existem no banco; ele é pensado para bootstrap.
