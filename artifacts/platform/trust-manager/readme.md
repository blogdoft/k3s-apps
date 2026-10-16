# trust-manager

Distribui a CA privada do cluster (`home-arpa-ca`) para ConfigMaps em todos os namespaces, mantendo
cópias sincronizadas quando a CA muda. Evita copiar o certificado à mão para cada aplicação.

Chart: `jetstack/trust-manager`, versão `v0.25.0` (compatível com o cert-manager `v1.19.x`).

## Como funciona

- O chart (`manifests/apply.yaml`) instala o controlador, o CRD `Bundle` e o webhook (cujo certificado
  vem do próprio cert-manager). Roda no namespace `cert-manager`, que é o *trust namespace*: é de lá que
  ele lê o Secret `home-arpa-ca`.
- O `Bundle` `home-arpa-ca` (`manifests/10-bundle-home-arpa-ca.yaml`) lê **só `tls.crt`** desse Secret e
  grava um ConfigMap `home-arpa-ca` (chave `home-arpa-ca.crt`) em cada namespace. A chave privada
  (`tls.key`) não é lida nem copiada.
- O trust-manager só tem permissão para ler Secrets do namespace `cert-manager` e não escreve Secrets.

## Usar a CA em um workload

Monte o ConfigMap e inclua o diretório em `SSL_CERT_DIR` (exemplo para .NET, mantendo as CAs do sistema):

```yaml
env:
  - name: SSL_CERT_DIR
    value: /etc/ssl/certs:/etc/ssl/home-arpa
volumeMounts:
  - name: home-arpa-ca
    mountPath: /etc/ssl/home-arpa
    readOnly: true
volumes:
  - name: home-arpa-ca
    configMap:
      name: home-arpa-ca
```

O trust-manager atualiza o ConfigMap, mas o processo só relê os certificados ao reiniciar. Combine com o
**Reloader** (`artifacts/platform/reloader`) anotando o Deployment com `reloader.stakater.com/auto: "true"`.

## Renderizar o template

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm template trust-manager jetstack/trust-manager \
  --version v0.25.0 \
  -n cert-manager \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

O `Bundle` fica fora do chart (`10-bundle-home-arpa-ca.yaml`) e **não** é gerado pelo `helm template`.

## Verificar

```sh
kubectl get bundle home-arpa-ca
kubectl get configmap home-arpa-ca -n code-brain -o jsonpath='{.data.home-arpa-ca\.crt}' | openssl x509 -noout -subject
```

## Cuidados

- Se a CA for **regenerada** (ver o lembrete em `CLAUDE.md`), o trust-manager atualiza os ConfigMaps
  sozinho, e o Reloader reinicia quem os monta. Os passos manuais de nós, runners e navegadores
  continuam necessários.
- Habilitar `defaultPackage` só é preciso para Bundles com `useDefaultCAs`; aqui está desligado.
