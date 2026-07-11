# harbor-stack — self-managed Harbor on k3s via ArgoCD

This directory is a Tidan **umbrella Helm chart** that deploys [Harbor](https://goharbor.io)
(the open-source container registry) onto the shared k3s cluster. It follows the same
GitOps convention as the other services in `Both/` (cosmic, realmhub, arcana): an ArgoCD
`Application` points at this repo's `k8s/` path and renders the chart with
`values.yaml` + `values.<env>.yaml`.

Harbor is **fully self-managed** — ArgoCD owns the entire stack (core, portal, registry,
jobservice, nginx, internal Postgres, internal Redis, Trivy scanner) and continuously
reconciles it from git.

## Layout

```
k8s/
├── Chart.yaml                 # umbrella chart (name: harbor-stack)
├── charts/
│   └── harbor-1.19.1.tgz      # VENDORED upstream goharbor chart (Harbor 2.15.1)
├── values.yaml                # base values (dev-ready; all under the `harbor:` key)
├── values.development.yaml    # dev overlay
├── values.production.yaml     # prod overlay (host, TLS secret, bigger PVCs, HA replicas)
├── secret.example.yaml        # how to create the required admin/secretKey Secret
└── templates/NOTES.txt
```

### Why the subchart is vendored (and not in `dependencies:`)

The upstream `harbor` chart is committed as a packaged tgz in `charts/`. It is **deliberately
not** listed under `dependencies:` in `Chart.yaml`. Helm auto-loads any chart found in
`charts/`, so `helm template` (and therefore ArgoCD's repo-server) renders it with **no
network access and no `Chart.lock`** at sync time. This keeps the deployment self-contained.

Trade-off: `helm lint` will print
`chart metadata is missing these dependencies: harbor`. That is expected and harmless —
`helm template` and ArgoCD do not enforce it. Verify with a render instead of a lint:

```bash
helm template harbor . -f values.yaml -f values.development.yaml --namespace harbor-development
```

## One-time prerequisites (per environment)

Harbor needs an admin password and a 16-char encryption key. They are **never** in git;
the chart reads them from a `harbor-core-secret` Secret via `existingSecretAdminPassword`
/ `existingSecretSecretKey`. Create it before (or at) first sync — see
[`secret.example.yaml`](./secret.example.yaml):

```bash
kubectl create namespace harbor-development
kubectl -n harbor-development create secret generic harbor-core-secret \
  --from-literal=HARBOR_ADMIN_PASSWORD="$(openssl rand -base64 24)" \
  --from-literal=secretKey="$(openssl rand -hex 8)"   # exactly 16 chars
```

(The ArgoCD Application sets `ignoreDifferences` on Secret `/data`, so ArgoCD won't fight
Harbor over the secrets it auto-generates on first install.)

## How it's wired into ArgoCD

The app-of-apps repo registers two Applications (`Both/Arcana-Argocd-Apps`):

| Env  | File                                   | Namespace            | Branch        | Host                      |
|------|----------------------------------------|----------------------|---------------|---------------------------|
| dev  | `dev/harbor-dev.yaml`                  | `harbor-development` | `development` | `harbor-dev.rrobinson.me` |
| prod | `prod/harbor-prod.yaml`                | `harbor-production`  | `main`        | `harbor.rrobinson.me`     |

Both point at `git@github.com:jojobobby/harbor.git`, `path: k8s`, releaseName `harbor`.

## Deploy / go-live checklist

1. Push this Harbor fork to `git@github.com:jojobobby/harbor.git` with both a `main`
   and a `development` branch (the k8s/ chart must be on both).
2. Give ArgoCD read access to the repo (deploy key / repo credential), same as the other
   private service repos.
3. Create the `harbor-core-secret` Secret in `harbor-development` and `harbor-production`
   (see above).
4. Ensure DNS `harbor-dev.rrobinson.me` and `harbor.rrobinson.me` resolve to the haproxy
   ingress, so cert-manager (letsencrypt-prod) can issue the TLS certs.
5. Commit & push `dev/harbor-dev.yaml` + `prod/harbor-prod.yaml` in `Arcana-Argocd-Apps`.
   The app-of-apps recurses `dev/` and `prod/`, so it auto-registers Harbor.
6. Watch ArgoCD sync; log in at the host as `admin` with the password from the Secret.

## Upgrading Harbor

1. Download the new chart from <https://helm.goharbor.io> into `charts/` and delete the
   old tgz: `curl -sLo charts/harbor-<ver>.tgz https://helm.goharbor.io/harbor-<ver>.tgz`
   (note `-o charts/...` — plain `-O` from `k8s/` drops the tgz next to Chart.yaml where
   Helm ignores it, and the whole Harbor stack silently vanishes from the render).
2. Bump `appVersion` in `Chart.yaml` to the new Harbor app version.
3. Diff the new upstream `values.yaml` against ours and reconcile any renamed/removed keys.
4. Re-render with `helm template` to confirm, commit, push — ArgoCD rolls it out.
