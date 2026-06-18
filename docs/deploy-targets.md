# Deploy Targets

All templates support two deploy targets: **GHCR** (GitHub Container Registry) and **VPS via SSH**.

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

## Choosing a target

| Scenario | Recommended target |
|----------|--------------------|
| Just want image versioning | GHCR only |
| Self-hosted server | VPS SSH |
| Both image registry + deploy | Both (default) |
| Testing / no server yet | `none` (via `workflow_dispatch` input) |
