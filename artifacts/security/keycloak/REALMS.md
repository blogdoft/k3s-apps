# Bootstrap de realms/clients no Keycloak (GitOps)

Este setup faz o Keycloak iniciar com `--import-realm` e carregar os arquivos JSON presentes em `/opt/keycloak/data/import`.
No Kubernetes, isso é alimentado pelo ConfigMap `keycloak-realm-import`.

## Tema de login Blog do FT

O tema `blogdoft` é entregue pelo ConfigMap
`keycloak-blogdoft-login-theme` e montado em
`/opt/keycloak/themes/blogdoft/login`. Ele estende o tema moderno
`keycloak.v2`, mantendo os templates e fluxos nativos do Keycloak, e aplica
o logo oficial, preto, vermelho e tons claros da identidade do Blog do FT.

Os exports versionados já definem `"loginTheme": "blogdoft"`. Como o
`--import-realm` não atualiza realms já existentes, para aplicar o tema a um
realm em uso selecione **Blog do FT** em **Realm settings > Themes > Login
theme** no Admin Console (ou atualize esse atributo pela Admin API). A adição
do volume reinicia o StatefulSet na primeira sincronização do Argo CD.

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

## Chave USB FIDO2 / PicoKey (WebAuthn como 2FA)

Os exports de `k8s` e `master` deixam o `WebAuthn Authenticator` como
`ALTERNATIVE` dentro de `Browser - Conditional 2FA`. Assim, uma chave FIDO2,
como a PicoKey, é um segundo fator opcional: depois de cadastrada, ela pode ser
usada no lugar do TOTP; usuários sem uma credencial WebAuthn continuam usando a
senha normalmente. O fluxo passwordless não é habilitado por esta configuração,
pois ele exige credencial descobrível e verificação do usuário (PIN), que nem
toda chave USB oferece.

Para cadastrar uma chave para um usuário existente:

1. Acesse o Admin Console em `https://keycloak.home.arpa/admin/` e selecione o
   realm apropriado (`k8s` para os aplicativos; `master` somente para a conta
   administrativa).
2. Em **Authentication > Required actions**, confirme que **Webauthn Register**
   está habilitado. Os exports já o deixam habilitado, mas não como ação padrão.
3. Em **Users > <usuário>**, use **Reset actions** e adicione
   **Webauthn Register**. No próximo login, conecte a PicoKey e confirme o
   toque (e o PIN, se o navegador/chave o solicitar). Dê um rótulo claro à
   credencial, por exemplo `PicoKey USB`.
4. Teste em uma janela anônima: usuário e senha primeiro; depois a chave. Se o
   usuário também tiver TOTP, use **Try another way** para alternar entre os
   fatores.

O `RP ID` permanece vazio de propósito: com `KC_HOSTNAME` definido como
`https://keycloak.home.arpa`, o Keycloak usa `keycloak.home.arpa`. Não acesse o
Keycloak por IP, por uma URL alternativa ou sem HTTPS durante o cadastro: o
WebAuthn vincula a credencial ao domínio de origem. Mantenha ao menos uma conta
administrativa e um método de recuperação testados antes de exigir WebAuthn para
todos os usuários.

Como a importação de realm só é aplicada no bootstrap, o ajuste dos JSONs não
altera realms que já existem no PostgreSQL. Para a instalação atual, altere o
mesmo passo no Admin Console: **Authentication > Flows > Browser > Browser -
Conditional 2FA > WebAuthn Authenticator > Alternative**. Depois, versionar o
export atualizado preserva a configuração para recriações futuras.
