# Strata — Deployment

Kubernetes manifests for **Strata** (the in-cluster Kubernetes database browser),
managed with [Kustomize](https://kustomize.io/) and delivered via
[ArgoCD](https://argo-cd.readthedocs.io/) (GitOps).

There is a **single environment** — `main`. Every change to this repo's `main`
branch is reconciled onto the cluster automatically; there is no staging/prod
split and no manual promotion step.

## How it works

```
push to app repo (main)
   → CI builds + tests + pushes image to ghcr.io/janne6565/strata-{backend,frontend}
   → CI runs `kustomize edit set image` in overlays/main and pushes to this repo
   → ArgoCD detects the commit and syncs the cluster
```

No manual `kubectl apply` is needed for normal deployments.

## Layout

```
base/                  shared manifests
  backend/             Deployment, Service, ServiceAccount, RBAC, ConfigMap
  frontend/            Deployment, Service (static SPA via nginx)
  postgres/            StatefulSet + Service (state DB)
overlays/
  main/                the only environment: namespace `strata`, Ingress, image tags
argocd/
  main-app.yaml        ArgoCD Application (auto-sync, self-heal, prune)
docs/
  secrets-required.md  secrets to create by hand before first sync
```

## Strata-specific notes

- **Cluster access:** the backend runs under a dedicated `strata-backend`
  ServiceAccount bound to a scoped `ClusterRole` (`base/backend/rbac.yaml`) so the
  discovery engine can read workloads, services, and (sensitively) secrets for
  credential resolution. No `cluster-admin`, no `*` verbs.
- **No Actuator:** the backend has no Actuator on the classpath, so probes are
  TCP-socket probes on `:8080`. Add `spring-boot-starter-actuator` to switch to
  proper `httpGet` health probes.
- **Migrations:** Flyway runs at backend startup (`ddl-auto: validate`). Keep
  migrations backward-compatible so rolling updates roll forward safely.

## First-time setup (new cluster)

1. Install ArgoCD and point it at this repo.
2. Create the required secrets — see [`docs/secrets-required.md`](docs/secrets-required.md).
3. Apply the ArgoCD Application:

   ```bash
   kubectl apply -f argocd/main-app.yaml
   ```

   ArgoCD creates the `strata` namespace and deploys everything.

DNS for `strata.jannekeipert.de` must point at the ingress controller; TLS is
issued automatically by cert-manager (`letsencrypt-prod`).

## Related repositories

| Repository | Description |
|---|---|
| `strata-backend` | Spring Boot backend — builds and pushes the backend image |
| `strata-frontend` | React + Vite frontend — builds and pushes the frontend image |
