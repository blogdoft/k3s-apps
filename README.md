# k3s-apps

This repository contains the **desired state** (GitOps) of the services running in my **k3s** cluster, all managed by **Argo CD**.
The idea is straightforward: everything that is running in the cluster is declared here, and Argo CD continuously reconciles the cluster with this repository.

The structure is organized as follows:

* `platform/`: **Argo CD Applications**, including the *root app* (App of Apps pattern)
* `artifacts/`: raw Kubernetes manifests and **Helm values** per service

---

## Deployed services and URLs

### Longhorn (distributed storage)

* **URL:** `https://longhorn.home.arpa/`
* Installed via the official Helm chart, with custom values and additional manifests stored in this repository.

### OpenBao (Vault-compatible secrets management)

* **URL:** `https://openbao.home.arpa/ui`
* UI enabled and exposed via Ingress.
* TLS is terminated at the Ingress level (internal TLS disabled in the chart).

### Flagr (feature flags)

* **URL:** `https://flagr.home.arpa/`
* Exposed via Ingress with HTTPS redirection.
* Health check endpoint:
  `GET /api/v1/health`

### Redis

* **Access:** `192.168.1.212:6379` (Service type `LoadBalancer`)
* Persistent storage backed by a PVC using the `longhorn-fast` StorageClass.

---

## Cluster prerequisites

Before applying anything from this repository, the cluster must meet the following requirements:

1. **Argo CD installed** in the `argocd` namespace.
2. **SSH access to this repository** configured in Argo CD
   (`git@github.com:blogdoft/k3s-apps.git`).
3. **Traefik as the Ingress Controller**
   (Ingress manifests rely on `ingressClassName: traefik` and Traefik annotations).
4. **Wildcard TLS certificate**
   A Secret named `wildcard-home-arpa` must exist in the namespaces where HTTPS is required (Longhorn, OpenBao, Flagr).
5. **Internal DNS** resolving `*.home.arpa` to the Traefik entrypoint (Ingress / LoadBalancer IP).
6. **Longhorn disk and node labeling**
   This repo defines a custom StorageClass `longhorn-fast` with:

   * `numberOfReplicas: "1"`
   * `diskSelector: ssd`
   * `nodeSelector: ssd`
     Nodes and disks must be labeled accordingly.

---

## Applying everything (GitOps – App of Apps)

The main entry point is the **root application** defined in:

```
platform/app-plataform.yaml
```

This application points back to the `platform/` directory in this repository and automatically creates and manages all child applications (Longhorn, OpenBao, Flagr, Redis).

Recommended flow:

1. Ensure Argo CD can access the repository via SSH.
2. Apply the root application:

```bash
kubectl apply -n argocd -f platform/app-plataform.yaml
```

3. Argo CD will automatically create and synchronize all child applications.
   All of them use:

   * automated sync
   * `prune: true`
   * `selfHeal: true`
   * `CreateNamespace: true`

---

## 🟦 Argo CD Sync Waves (Deployment Order)

This repository relies on **Argo CD sync waves** to guarantee a deterministic and safe deployment order, both **between Applications (App of Apps)** and **inside each Application**.

Sync waves are defined using the annotation:

```yaml
argocd.argoproj.io/sync-wave: "<integer>"
```

Lower numbers are applied first. Resources without an explicit wave default to `0`.

---

### Application-level waves (App of Apps)

The root application (`platform/app-plataform.yaml`) manages multiple child Applications.
Their synchronization order is explicitly controlled using sync waves:

| Wave | Application  | Purpose                                      |
| ---- | ------------ | -------------------------------------------- |
| `0`  | **longhorn** | Storage foundation (required by PVCs)        |
| `10` | **openbao**  | Secrets management (uses persistent storage) |
| `20` | **redis**    | Stateful workload backed by Longhorn         |
| `30` | **flagr**    | Application layer                            |

This ensures that **infrastructure and platform services are always ready before application workloads**.

---

### Resource-level waves (inside Applications)

Within each Application, sync waves are also used to express **hard dependencies between Kubernetes resources**.

#### Standard convention used in this repository

| Wave  | Resource type                                                       |
| ----- | ------------------------------------------------------------------- |
| `-10` | Namespaces, CRDs                                                    |
| `0`   | Base resources (PVCs, ServiceAccounts, ConfigMaps, Secrets)         |
| `10`  | Workloads (Deployments, StatefulSets, Jobs)                         |
| `20`  | Exposure (Services, Ingress, HTTPRoutes)                            |
| `30+` | Integrations and post-install resources                             |
| `50`  | Post-Helm add-ons (e.g. StorageClasses created after chart install) |

#### Examples

* **Redis**

  * Namespace → PVC → Deployment → Service
* **Flagr**

  * Deployment → Service → Traefik Middleware → Ingress
* **Longhorn**

  * Helm chart → post-install `StorageClass`

This approach avoids race conditions, improves observability in the Argo CD UI, and makes deployment intent explicit.

---

### Design principles

* Sync waves are used **only when real dependencies exist**
* Numbers are spaced (`10`, `20`, `30`) to allow future insertion
* Waves express **contracts**, not cosmetic ordering
* Hooks (`PreSync` / `PostSync`) are preferred for one-off tasks such as migrations or seeding

---

## Application details

### `platform/app-longhorn.yaml`

* Installs Longhorn via Helm.
* Applies additional manifests from `artifacts/longhorn/manifests`.
* Defines the `longhorn-fast` StorageClass after installation.

### `platform/app-openbao.yaml`

* Installs OpenBao via Helm.
* Uses versioned values stored in this repository.
* Exposes the UI at `openbao.home.arpa`.

### `platform/app-flagr.yaml`

* Deploys Flagr using raw manifests from `artifacts/flagr/deployment.yaml`.
* Exposes the service at `flagr.home.arpa` with HTTPS enforcement.

### `platform/app-redis.yaml`

* Deploys Redis using manifests from `artifacts/redis/apply.yaml`.
* Exposes Redis via a `LoadBalancer` with a fixed IP (`192.168.1.212`).

---

## Secrets and sensitive configuration

### Flagr – PostgreSQL credentials

The Flagr deployment expects a Secret named `postgres-credentials` in the `flagr` namespace with the following keys:

* `POSTGRES_USER`
* `POSTGRES_PASSWORD`
* `POSTGRES_HOST`
* `POSTGRES_PORT`
* `POSTGRES_DB`

**Recommendation:**
Do not commit real values to Git. Prefer solutions such as OpenBao, External Secrets, Sealed Secrets, or SOPS.

### Redis – hardcoded password (important)

Redis is currently configured with `--requirepass` directly in the manifest, which means the password is **stored in Git**.

This is not recommended. A better approach would be:

* Move the password to a Kubernetes Secret.
* Inject it via environment variables or command arguments.

Additionally, `ALLOW_EMPTY_PASSWORD="yes"` is set, which is usually unnecessary and can be misleading from a security standpoint.

---

## Operations and quick troubleshooting

* List Argo CD applications:

  ```bash
  kubectl get applications -n argocd
  ```

* List all Ingress resources:

  ```bash
  kubectl get ingress -A
  ```

* Check Flagr health:

  ```bash
  curl -k https://flagr.home.arpa/api/v1/health
  ```

* Inspect Redis service:

  ```bash
  kubectl -n redis-server get svc redis-server
  ```

---

## Suggested improvements / roadmap

* Remove hardcoded Redis credentials and standardize secret management.
* Ensure the `wildcard-home-arpa` TLS Secret exists in all required namespaces.
* Document and standardize node and disk labeling for Longhorn (`ssd` selectors).
* Optionally add a small `README.md` per service under `artifacts/<service>/` describing configuration, backup, and restore procedures.
