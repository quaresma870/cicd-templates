# Deploy Targets

All templates support **GHCR** (GitHub Container Registry), **VPS via SSH**,
**Fly.io**, **Kubernetes**, and **AWS ECS**.

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

## AWS ECS

`deploy_target: ecs` registers a new revision of an existing ECS task
definition family with the new image, then updates the service and waits
for it to stabilize — it does **not** create the cluster, service, or task
definition. Same scope limit as `k8s`: provisioning (VPC, cluster, service,
task definition, IAM roles) is environment-specific and assumed already
done; rolling out a new image onto an existing service is the one part
this repo's pipeline can own.

Authenticates with **static AWS access keys**, not OIDC federation — a
deliberate choice for this repo, not a placeholder pending a future change.

### Prerequisites

1. An ECS cluster, service, and task definition family already set up, with
   a container name matching `APP_NAME` (or `image_name` for the reusable
   workflows) — the task definition family name must also match `APP_NAME`.
2. An IAM user with `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` secrets,
   scoped to just what `deploy-ecs` needs — see
   [secrets-setup.md](secrets-setup.md#deploying-to-aws-ecs).
3. `AWS_REGION` and `ECS_CLUSTER` variables (both required); `ECS_SERVICE`
   defaults to `APP_NAME` if unset.
4. If pulling from a private GHCR repository (the default for every
   template here), the task definition's container needs
   `repositoryCredentials` already configured pointing at a Secrets
   Manager secret with a GHCR PAT — `deploy-ecs` only overrides the image
   URI, it doesn't touch that field.

### How deploy works

1. `aws-actions/configure-aws-credentials` authenticates with the static
   keys
2. `aws ecs describe-task-definition --task-definition <family>` downloads
   the current task definition as JSON
3. `aws-actions/amazon-ecs-render-task-definition` patches in the new image
   URI for the matching container name
4. `aws-actions/amazon-ecs-deploy-task-definition` registers the rendered
   definition as a new revision, updates the service, and (with
   `wait-for-service-stability: true`) blocks until the rollout stabilizes

Like `fly`/`k8s`, `ecs` is always selected on its own — it isn't folded
into `deploy_target: both`.

---

## Choosing a target

| Scenario | Recommended target |
|----------|--------------------|
| Just want image versioning | GHCR only |
| Self-hosted server | VPS SSH |
| Both image registry + deploy | Both (default) |
| No server to manage | Fly.io |
| Already running a Kubernetes cluster | Kubernetes |
| Already running on ECS | AWS ECS |
| Testing / no server yet | `none` (via `workflow_dispatch` input) |
