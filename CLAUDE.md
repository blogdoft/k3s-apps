# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

GitOps source of truth for a home-lab K3s cluster, deployed by ArgoCD using the
App-of-Apps pattern. There is no application code and no build/test/lint tooling here —
this repo *is* the desired cluster state. "Making a change" means editing YAML and, for
Helm-based apps, re-rendering manifests locally with `helm template` before committing.

Nothing in this repo talks to the live cluster directly from CI; ArgoCD (running in the
cluster, pointed at `git@github.com:blogdoft/k3s-apps.git`) polls this repo and reconciles.
There are no GitHub Actions/CI pipelines — validation is "does `helm template` render" and
"does ArgoCD sync cleanly," checked manually against the real cluster.

## Repository layout

```
bootstrap/
  root-app.yaml           # the ONE resource applied manually (kubectl apply -n argocd -f bootstrap/root-app.yaml)
  projects/*.yaml          # ArgoCD AppProject defs — one per domain (platform/security/apps/databases/observability/home-lab)
  applications/*.yaml      # ArgoCD Application defs — one per deployed service, references artifacts/<domain>/<app>/...
artifacts/
  platform/    security/    apps/    databases/    observability/    home-lab/
    <app>/
      manifests/apply.yaml       # rendered/raw K8s manifests — THIS is what ArgoCD actually syncs
      helm-values/values.yaml    # only present for Helm-based apps; NOT read by ArgoCD directly
```

`bootstrap/root-app.yaml` is an ArgoCD Application pointed at `bootstrap/` with
`directory.recurse: true`, so it auto-discovers every file under `bootstrap/projects/`
and `bootstrap/applications/`. Adding a new project or application means adding a YAML
file there — no registration step elsewhere.

Each `bootstrap/applications/<name>.yaml` Application's `spec.source.path` points at an
`artifacts/<domain>/<app>/manifests` (or `manifests/apply.yaml`'s directory) — never at
`helm-values/`. `helm-values/` exists purely so a chart can be re-rendered later; ArgoCD
never reads it.

## The Helm pattern (important — most apps use this)

For chart-based apps (alloy, grafana's datasources aren't but most observability apps,
loki, tempo, prometheus, otel-collector, minio, open-webui, longhorn, rancher, openbao),
this repo does **not** let ArgoCD talk to Helm. Instead, charts are rendered locally into
plain manifests and only the rendered output is committed and synced:

```sh
helm repo add <repo-name> <repo-url>
helm repo update

helm template <release-name> <repo-name>/<chart> \
  --version <chart-version> \
  -n <namespace> \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repeat this exact command (bumping `--version` on upgrades) any time
`helm-values/values.yaml` changes, and commit the diff in `manifests/apply.yaml` alongside
it. Check the app's `artifacts/<domain>/<app>/readme.md` for the exact chart repo/version/
namespace used — each documents its own render command, chart gotchas (default scrape jobs
that need explicit disabling, required `fullnameOverride`, missing `image.repository`,
etc.) and anything deliberately left unconfigured (secrets, HA, TLS). Read it before
touching that app.

A few apps are hand-written manifests instead of a rendered chart (e.g. grafana's
Deployment/dashboards/alerting ConfigMaps, flagr, kafka-ui, stremio) — for those,
edit `manifests/` directly.

## Secrets

This repo deliberately holds **no real secret values** for most apps — Secrets are
created manually on the cluster before first apply (see each app's readme.md for the
exact `kubectl create secret ...` invocation and the keys the manifest expects via
`secretKeyRef`, e.g. `postgres-credentials` for flagr, `grafana-admin` for grafana,
`minio-credentials` for minio, `open-webui-database` for open-webui). When adding a new
app that needs credentials, follow this convention (external creation, referenced by name
in the manifest) rather than committing values — the Redis app is a known exception
(password inline in the manifest) called out as a documented risk, not a pattern to copy.

## Sync waves — deployment ordering

Two independent layers of `argocd.argoproj.io/sync-wave` annotations control ordering;
get these wrong and dependent resources race:

1. **Application-level** (in `bootstrap/applications/*.yaml` and
   `bootstrap/projects/*.yaml`): controls the order whole apps/projects come up in
   (e.g. cert-manager before longhorn/openbao before redis before keycloak before
   flagr/kafka-ui). Platform/security projects sync before apps/observability/databases
   projects.
2. **Resource-level** (inside a single app's `manifests/apply.yaml` or hand-written
   manifests): namespaces/CRDs (`-10`) → base resources like PVCs/ConfigMaps/Secrets (`0`)
   → workloads (`10`) → Services/Ingress (`20`) → integrations (`30+`) → post-Helm add-ons
   like a StorageClass created after chart install (`50`).

Only add a wave annotation where a real dependency exists; leave it off (defaults to `0`)
for anything that doesn't need ordering. See `README.md`'s "Sync Waves" section for the
full table of current wave assignments before changing one.

## Ingress / TLS conventions

- `ingressClassName: traefik`, hosts are `<app>.home.arpa`.
- Don't reference a TLS secret per-Ingress: a wildcard cert (`wildcard-home-arpa`,
  cert-manager, in `kube-system`) is registered as Traefik's cluster-wide default via a
  TLSStore (`artifacts/platform/cert-manager/manifests/20-traefik-tlsstore-default.yaml`),
  so any `*.home.arpa` Ingress gets HTTPS automatically.
- The wildcard only covers one subdomain level — a host needs a form like
  `minio-api.home.arpa`, not `s3.minio.home.arpa`.

## Commit conventions

- Write all commit messages in English, following
  [Conventional Commits](https://www.conventionalcommits.org/) (`type(scope): summary`,
  e.g. `feat(flagr): enable prometheus metrics`) — matches the existing commit history's
  type/scope structure, just in English instead of Portuguese.
- Every commit needs a body explaining *what changed and what effect it has* (e.g. new
  behavior exposed, resource now restarted, dependency now required) — not just a
  restatement of the diff. A bare one-line subject is not enough.

## Working in this repo

- After editing any manifest or values file, sanity-check the YAML and, for Helm apps,
  regenerate `manifests/apply.yaml` with the exact command from that app's `readme.md`
  rather than hand-editing the rendered output (hand edits get silently discarded on the
  next re-render).
- There's no automated deploy from this session — changes only take effect once committed
  and ArgoCD (in the actual cluster) syncs. Don't assume a change is "live" after editing.
- `README.md` has the authoritative list of deployed services, URLs, and troubleshooting
  commands (`kubectl get applications -n argocd`, etc.) — check it for details not
  duplicated here, and update it when adding/removing/moving an app so it stays accurate.
