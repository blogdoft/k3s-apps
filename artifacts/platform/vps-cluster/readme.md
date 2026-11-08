# vps-cluster — registering the VPS k3s cluster in Argo CD

The VPS (`<vpn-vps-ip>` on the WireGuard network) runs its own, separate k3s cluster.
Neither the VPS nor morgul join each other as nodes; Argo CD on morgul only manages the
VPS cluster as a remote destination over the WireGuard tunnel.

- API endpoint: `https://<vpn-vps-ip>:6443` (reachable only through `wg0`; ufw on the VPS
  allows 6443/tcp from `<vpn-ip-range>/24` on `wg0` and nothing else on that interface).
- The API serving certificate has no `<vpn-vps-ip>` SAN, so the cluster Secret sets
  `tlsClientConfig.serverName: kubernetes` (a SAN it does have) instead of restarting k3s.
- The `argocd-manager` ServiceAccount is `cluster-admin` on the VPS. Treat the token as
  highly sensitive. Nothing in this repo contains it.

## 1. On the VPS — create the ServiceAccount and token

```sh
scp vps-argocd-manager.yaml <vps-user-name>@<vps-external-ip>:/tmp/
ssh <vps-user-name>@<vps-external-ip> 'sudo k3s kubectl apply -f /tmp/vps-argocd-manager.yaml && rm /tmp/vps-argocd-manager.yaml'
```

## 2. On morgul — create the cluster Secret (never committed)

```sh
TOKEN=$(ssh <vps-user-name>@<vps-external-ip> \
  "sudo k3s kubectl -n kube-system get secret argocd-manager-token -o jsonpath='{.data.token}' | base64 -d")
CA=$(ssh <vps-user-name>@<vps-external-ip> \
  "sudo k3s kubectl -n kube-system get secret argocd-manager-token -o jsonpath='{.data.ca\.crt}'")

kubectl -n argocd create secret generic cluster-vps \
  --from-literal=name=vps \
  --from-literal=server=https://<vpn-vps-ip>:6443 \
  --from-literal=config="{\"bearerToken\":\"$TOKEN\",\"tlsClientConfig\":{\"serverName\":\"kubernetes\",\"caData\":\"$CA\"}}"
kubectl -n argocd label secret cluster-vps argocd.argoproj.io/secret-type=cluster
```

Check: `kubectl -n argocd get secret cluster-vps` and, in the Argo CD UI,
Settings → Clusters → `vps` should be `Successful`.

## 3. Deploying to the VPS

Add the destination to the relevant `AppProject` (`bootstrap/projects/*.yaml`) as
`server: https://<vpn-vps-ip>:6443` (or `name: vps`), then create Applications with
`spec.destination.server: https://<vpn-vps-ip>:6443`.

## Rotation / removal

Delete the `argocd-manager-token` Secret on the VPS to revoke access, and delete
`cluster-vps` from `argocd` on morgul.
