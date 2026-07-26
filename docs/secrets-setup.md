# Secrets Setup

How to configure the required secrets and variables for each template.

## Adding secrets to a GitHub repo

Go to: **Settings → Secrets and variables → Actions → New repository secret**

---

## Common secrets (all deploy templates)

| Secret | Description | Example |
|--------|-------------|---------|
| `GHCR_TOKEN` | GitHub PAT with `packages:write` + `read:packages` | `ghp_xxx...` |
| `VPS_HOST` | Your server IP or domain | `192.168.1.1` or `myserver.com` |
| `VPS_USER` | SSH user on the server | `deploy` |
| `VPS_SSH_KEY` | Private SSH key — PEM format, full content | `-----BEGIN OPENSSH...` |
| `VPS_PORT` | SSH port | `22` |
| `FLY_API_TOKEN` | Only if `deploy_target` is `fly` — flyctl auth token | `fm2_xxx...` |
| `KUBE_CONFIG` | Only if `deploy_target` is `k8s` — base64-encoded kubeconfig | (base64 blob) |
| `AWS_ACCESS_KEY_ID` | Only if `deploy_target` is `ecs` | `AKIA...` |
| `AWS_SECRET_ACCESS_KEY` | Only if `deploy_target` is `ecs` | (secret key) |

### Generating a deploy SSH key

```bash
# Generate a dedicated deploy key (no passphrase)
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key -N ""

# Add the public key to your server
ssh-copy-id -i ~/.ssh/deploy_key.pub user@your-server

# Copy the private key content → paste as VPS_SSH_KEY secret
cat ~/.ssh/deploy_key
```

### Generating a GHCR token

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Generate new token with scopes: `write:packages`, `read:packages`, `delete:packages`
3. Paste the token as `GHCR_TOKEN` secret

### Deploying to Fly.io

The `fly` deploy target needs a `fly.toml` in the repo root — run `fly launch`
locally once to generate it (this also creates the Fly app itself). Then:

```bash
fly tokens create deploy -x 999999h   # or: flyctl auth token
```

Paste the resulting token as the `FLY_API_TOKEN` secret. Unlike `ghcr`/`vps`,
the `deploy-fly` job builds the image itself on Fly's remote builder straight
from the Dockerfile — it doesn't reuse the image already pushed to GHCR.

### Deploying to Kubernetes

The `k8s` deploy target assumes a Deployment already exists in the cluster —
`deploy-k8s` only rolls out a new image (`kubectl set image` +
`kubectl rollout status`), it doesn't apply manifests or create resources.
The Deployment's container name must match `APP_NAME` (or `image_name` for
the reusable workflows).

```bash
# Base64-encode your kubeconfig (a service-account-scoped one, not your
# personal admin config, is strongly recommended for CI use)
base64 -w0 ~/.kube/config   # macOS: base64 -i ~/.kube/config
```

Paste the output as the `KUBE_CONFIG` secret. Set the `KUBE_NAMESPACE`
variable if your Deployment isn't in the `default` namespace (or pass
`kube_namespace` for the reusable workflows).

### Deploying to AWS ECS

The `ecs` deploy target assumes an ECS service + task definition family
already exist — `deploy-ecs` only registers a new task definition revision
with the new image and updates the service, it doesn't create the
cluster, service, or task definition. The task definition's container name
must match `APP_NAME` (or `image_name` for the reusable workflows), and its
family name must also match `APP_NAME`.

Uses **static AWS access keys**, not OIDC federation — create an IAM user
scoped to just `ecs:DescribeTaskDefinition`, `ecs:RegisterTaskDefinition`,
and `ecs:UpdateService` on the specific cluster/service (plus
`iam:PassRole` for the task's execution/task roles), then generate an
access key for it. Paste the key ID and secret as `AWS_ACCESS_KEY_ID` /
`AWS_SECRET_ACCESS_KEY`. Set the `AWS_REGION` and `ECS_CLUSTER` variables
(both required, no sensible default); `ECS_SERVICE` defaults to `APP_NAME`
if unset.

If the task definition pulls the image from a private GHCR repository (the
default for every template here), its container definition needs
`repositoryCredentials` pointing at a Secrets Manager secret holding a
GHCR PAT — set that up once on the task definition itself; `deploy-ecs`
only overrides the image URI, it doesn't touch `repositoryCredentials`.

---

## Python template

| Secret/Variable | Type | Description |
|-----------------|------|-------------|
| `GHCR_TOKEN` | Secret | GHCR push token |
| `VPS_HOST` | Secret | Deploy server |
| `VPS_USER` | Secret | SSH user |
| `VPS_SSH_KEY` | Secret | SSH private key |
| `VPS_PORT` | Secret | SSH port |
| `APP_NAME` | Variable | Docker image name (e.g. `my-api`) |
| `DEPLOY_PATH` | Variable | Path on VPS (e.g. `/opt/my-api`) |
| `FLY_API_TOKEN` | Secret | Only if `deploy_target` is `fly` |
| `KUBE_CONFIG` | Secret | Only if `deploy_target` is `k8s` |
| `KUBE_NAMESPACE` | Variable | Only if `deploy_target` is `k8s` (default: `default`) |
| `AWS_ACCESS_KEY_ID` | Secret | Only if `deploy_target` is `ecs` |
| `AWS_SECRET_ACCESS_KEY` | Secret | Only if `deploy_target` is `ecs` |
| `AWS_REGION` | Variable | Only if `deploy_target` is `ecs` |
| `ECS_CLUSTER` | Variable | Only if `deploy_target` is `ecs` |
| `ECS_SERVICE` | Variable | Only if `deploy_target` is `ecs` (default: `APP_NAME`) |

---

## Go template

Same as Python — see the table above.

---

## Java template

Same as Python — see the table above.

---

## Rust template

Same as Python — see the table above.

---

## Node.js template

Same as Python, plus optionally:

| Secret | Type | Description |
|--------|------|-------------|
| `SEMGREP_APP_TOKEN` | Secret | For Semgrep cloud dashboard (optional) |

---

## Ansible template

| Secret/Variable | Type | Description |
|-----------------|------|-------------|
| `ANSIBLE_VAULT_PASSWORD` | Secret | Password for `ansible-vault` encrypted vars |
| `INVENTORY_HOST` | Secret | Target host IP/domain |
| `INVENTORY_USER` | Secret | SSH user for Ansible |
| `INVENTORY_SSH_KEY` | Secret | SSH private key |
| `PLAYBOOK` | Variable | Main playbook filename (e.g. `site.yml`) |

---

## Terraform template

| Secret/Variable | Type | Description |
|-----------------|------|-------------|
| `TF_API_TOKEN` | Secret | Terraform Cloud API token |
| `AWS_ACCESS_KEY_ID` | Secret | AWS key (if using AWS provider) |
| `AWS_SECRET_ACCESS_KEY` | Secret | AWS secret (if using AWS provider) |
| `TF_WORKING_DIR` | Variable | Directory with .tf files (default: `.`) |
| `TF_WORKSPACE` | Variable | Terraform workspace (default: `default`) |

---

## Security template

| Secret | Required | Description |
|--------|----------|-------------|
| `GHCR_TOKEN` | Optional | For container image scanning |
| `SLACK_WEBHOOK` | Optional | Slack notifications on scheduled scan failures |
| `SEMGREP_APP_TOKEN` | Optional | Semgrep cloud dashboard |
| `GITLEAKS_LICENSE` | Optional | Gitleaks Pro license (free tier works without it) |
