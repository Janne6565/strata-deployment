# Required Kubernetes Secrets

Apply these manually to the `strata` namespace before ArgoCD can deploy
successfully. **Never commit secret values to this repository.**

## namespace: strata

### `strata-backend-secrets`

Backend secrets, injected via `envFrom` and read by Spring through the
`${STRATA_*}` placeholders in `application.yaml`. Non-secret connection config
(DB URL/user, owner username) lives in the `strata-backend-config` ConfigMap
instead — only the values below are secret.

```bash
kubectl create secret generic strata-backend-secrets \
  --namespace=strata \
  --from-literal=STRATA_DB_PASSWORD=... \
  --from-literal=STRATA_OWNER_PASSWORD=... \
  --from-literal=STRATA_JWT_SECRET=... \
  --from-literal=STRATA_OAUTH_CLIENT_ID=... \
  --from-literal=STRATA_OAUTH_CLIENT_SECRET=...
```

- `STRATA_DB_PASSWORD` — must match `POSTGRES_PASSWORD` in `postgres-secrets`.
- `STRATA_OWNER_PASSWORD` — first-owner bootstrap password. Change it via the UI
  after the first login; it is only used while no OWNER user exists.
- `STRATA_JWT_SECRET` — long random string used to sign access tokens
  (e.g. `openssl rand -base64 48`). Rotating it invalidates all sessions.
- `STRATA_OAUTH_CLIENT_ID` — OAuth client id for "Login with Authentik". **Must match
  the value sealed into `authentik-secrets` as `STRATA_OAUTH_CLIENT_ID`** in
  `cluster-deployment/infrastructure/authentik-secret.sealed.yaml` (the Authentik
  `strata` provider reads it via `!Env`). Mismatched values break the OIDC flow.
- `STRATA_OAUTH_CLIENT_SECRET` — matching OAuth client secret; **must match
  `STRATA_OAUTH_CLIENT_SECRET` in `authentik-secrets`** likewise.

### `postgres-secrets`

```bash
kubectl create secret generic postgres-secrets \
  --namespace=strata \
  --from-literal=POSTGRES_PASSWORD=...
```

The `STRATA_DB_PASSWORD` in `strata-backend-secrets` must match this value.

### `ghcr-pull-secret` (only needed if the ghcr.io packages are private)

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace=strata \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<github-pat-with-read:packages>
```

If you make the `ghcr.io/janne6565/strata-*` packages public, you can drop the
`imagePullSecrets` references from the deployments and skip this secret.

## GitHub Actions secrets (set in each app repo)

| Secret | Repo(s) | Description |
|--------|---------|-------------|
| `DEPLOY_SSH_KEY` | backend, frontend | Private half of a write-scoped **SSH deploy key** registered on `strata-deployment`. CI uses it to check out the deployment repo, bump the image tag, and push. |

The deploy key's **public** half is added to this repo under
**Settings → Deploy keys** with *Allow write access* enabled. One key pair is
shared by both app repos (registered once here, private half stored as a secret
in each). Rotate by deleting the deploy key here and replacing the secret.

`GITHUB_TOKEN` (built in) is used to push images to GHCR — no extra secret
needed for that, just ensure **Settings → Actions → Workflow permissions** allows
package writes, or that the repo can publish to the `janne6565` GHCR namespace.
