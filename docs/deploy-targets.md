# Deploy Targets

All templates support **GHCR** (GitHub Container Registry), **VPS via SSH**,
**Fly.io**, and **Kubernetes**.

---

## GHCR (GitHub Container Registry)

Docker images are pushed to `ghcr.io/<owner>/<app-name>`.

### Setup

1. Add `GHCR_TOKEN` secret (see [secrets-setup.md](secrets-setup.md))
2. Set `APP_NAME` variable in repo settings
3. The pipeline pushes automatically on every merge to `main`

### Image tags

Each push generates multiple tags:

| Trigger | Tags produced |
|---------|---------------|
| Push to `main` | `main`, `sha-<commit>`, `latest` |
| Push to `dev` | `dev`, `sha-<commit>` |
| Git tag `v1.2.3` | `1.2.3`, `1.2`, `sha-<commit>` |

### Pulling the image on your server

```bash
echo "<GHCR_TOKEN>" | docker login ghcr.io -u <github-username> --password-stdin
docker pull ghcr.io/<owner>/<app-name>:latest
```

---

## VPS via SSH

The pipeline SSHs into your server and runs `docker compose up -d` with the new image.

### VPS prerequisites

Your server needs:

```bash
# Docker + Compose
curl -fsSL https://get.docker.com | sh
apt install docker-compose-plugin -y

# Deploy user (don't use root)
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
```

### Directory structure on the VPS

```
/opt/<app-name>/           ← DEPLOY_PATH variable
├── docker-compose.yml
└── .env                   ← production env vars
```

### Example `docker-compose.yml` on VPS

```yaml
services:
  app:
    image: ghcr.io/<owner>/<app-name>:latest
    restart: unless-stopped
    ports:
      - "8000:8000"
    env_file: .env
```

### How deploy works

1. Pipeline builds and pushes image to GHCR
2. SSH into VPS
3. `docker login ghcr.io` with GHCR token
4. `docker pull <image>:latest`
5. `docker compose up -d --no-build` — restarts only changed containers
6. `docker system prune -f` — cleans up old images

### Zero-downtime deploy (optional)

To avoid downtime, add a health check to your `docker-compose.yml`:

```yaml
services:
  app:
    image: ghcr.io/<owner>/<app-name>:latest
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
```

Docker Compose will wait for the health check before switching traffic.

---

## Fly.io

`deploy_target: fly` runs `flyctl deploy --remote-only`, which builds the
image on Fly's own remote builder straight from your repo's `Dockerfile` and
deploys it — it does **not** reuse the image the `build` job already pushed
to GHCR. That's a deliberate tradeoff: Fly.io has no clean native way to
authenticate to a *private* GHCR image at deploy time (a long-standing gap
per Fly's own community forum), so building directly from the Dockerfile
sidesteps that friction entirely, at the cost of building the image twice.

### Prerequisites

1. A `fly.toml` in the repo root — run `fly launch` locally once to generate
   it (this also creates the Fly app). The templates don't create this file
   for you.
2. A `FLY_API_TOKEN` secret — see [secrets-setup.md](secrets-setup.md#deploying-to-flyio).

### How deploy works

1. `superfly/flyctl-actions/setup-flyctl` installs the Fly CLI
2. `flyctl deploy --remote-only` builds from `Dockerfile` on Fly's builder
   and deploys, using `fly.toml` for app name, region, and service config

Unlike `vps`, `fly` isn't combined by `deploy_target: both` — it's always
selected on its own, since it doesn't share GHCR's pushed image the way
`ghcr`+`vps` do.

---

## Kubernetes

`deploy_target: k8s` runs `kubectl set image` against a Deployment that
already exists in your cluster, then waits on `kubectl rollout status` —
it does **not** apply manifests, create a Deployment, or manage any other
cluster resources. That's a deliberate scope limit: provisioning is
cluster/environment-specific (Helm chart, raw manifests, Kustomize, GitOps
tooling), while rolling out a new image tag onto an existing Deployment is
the one part that's the same regardless of how you provisioned it — same
shape as `vps` assuming `docker-compose.yml` is already on the server rather
than the pipeline creating it.

### Prerequisites

1. A Deployment already running in the cluster, with a container name
   matching `APP_NAME` (or `image_name` for the reusable workflows).
2. A `KUBE_CONFIG` secret (base64-encoded kubeconfig) — see
   [secrets-setup.md](secrets-setup.md#deploying-to-kubernetes). A
   service-account-scoped kubeconfig limited to `get`/`patch` on that one
   Deployment's namespace is strongly recommended over a full admin config.
3. Optionally, a `KUBE_NAMESPACE` variable (or `kube_namespace` input) if
   the Deployment isn't in the `default` namespace.

### How deploy works

1. `azure/setup-kubectl` installs the `kubectl` CLI
2. The kubeconfig secret is base64-decoded to `~/.kube/config`
3. `kubectl set image deployment/<app> <app>=<new-image> -n <namespace>`
   triggers the rollout
4. `kubectl rollout status ... --timeout=180s` waits for it to finish (fails
   the job if the rollout doesn't complete in time)
5. The kubeconfig file is deleted from the runner regardless of outcome

Like `fly`, `k8s` is always selected on its own — it isn't folded into
`deploy_target: both`.

---

## Choosing a target

| Scenario | Recommended target |
|----------|--------------------|
| Just want image versioning | GHCR only |
| Self-hosted server | VPS SSH |
| Both image registry + deploy | Both (default) |
| No server to manage | Fly.io |
| Already running a cluster | Kubernetes |
| Testing / no server yet | `none` (via `workflow_dispatch` input) |
