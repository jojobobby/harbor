# Tidan deployment of this Harbor fork

This is a fork of [`goharbor/harbor`](https://github.com/goharbor/harbor) used to run a
self-hosted container registry for the Tidan/Realm projects, deployed to the shared k3s
cluster via **ArgoCD** (GitOps), exactly like the other services in the `Both/` workspace.

- `upstream` remote → `https://github.com/goharbor/harbor.git` (pull updates from here)
- `origin` remote   → `git@github.com:jojobobby/harbor.git` (our fork; ArgoCD pulls this)

## What we added on top of upstream

Everything Tidan-specific lives in **[`k8s/`](./k8s/)** — a self-contained umbrella Helm
chart (`harbor-stack`) that vendors the official Harbor chart and configures it for our
cluster (haproxy ingress, cert-manager, `harbor[-dev].rrobinson.me`, namespaces, secrets).
Nothing in the upstream Go source is modified, so rebasing on new Harbor releases is clean.

See **[`k8s/README.md`](./k8s/README.md)** for the full deploy/go-live checklist, the
per-environment hosts/namespaces, and how to upgrade the vendored chart.

## ArgoCD registration

The ArgoCD `Application` manifests live in the app-of-apps repo, not here:

- `Both/Arcana-Argocd-Apps/dev/harbor-dev.yaml`   → namespace `harbor-development`
- `Both/Arcana-Argocd-Apps/prod/harbor-prod.yaml` → namespace `harbor-production`

Both reference this repo at `path: k8s`, branches `development` (dev) / `main` (prod).
