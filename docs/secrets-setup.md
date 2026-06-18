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
