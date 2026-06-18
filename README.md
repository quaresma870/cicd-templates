# ⚙️ CI/CD Templates

[![Validate Templates](https://github.com/quaresma870/cicd-templates/actions/workflows/validate.yml/badge.svg?branch=main)](https://github.com/quaresma870/cicd-templates/actions/workflows/validate.yml)
![Node.js](https://img.shields.io/badge/GitHub%20Actions-Node.js%2024-brightgreen?logo=nodedotjs&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

A collection of production-ready GitHub Actions CI/CD pipeline templates covering multiple stacks. Each template includes lint, test, security scan, Docker build, and deploy stages (GHCR + VPS SSH).

---

## Templates

| Template | Stack | Stages |
|----------|-------|--------|
| [`python/`](templates/python/) | Python / FastAPI / Flask | Lint → Test → Security → Build → Deploy |
| [`nodejs/`](templates/nodejs/) | Node.js / Express / Next.js | Lint → Test → Security → Build → Deploy |
| [`generic/`](templates/generic/) | Any language | Validate → Test → Security → Build → Deploy |
| [`docker-only/`](templates/docker-only/) | Docker images | Lint → Security → Build → Push → Deploy |
| [`ansible/`](templates/ansible/) | Ansible playbooks | Lint → Syntax → Security → Dry Run → Deploy |
| [`terraform/`](templates/terraform/) | Terraform IaC | Format → Validate → Security → Plan → Apply |
| [`security/`](templates/security/) | Any repo | Secrets → Deps → SAST → Container → SBOM |

---

## How to use

1. Copy the relevant `ci.yml` to your repo's `.github/workflows/` directory
2. Configure the required secrets and variables (see each template's README or [docs/secrets-setup.md](docs/secrets-setup.md))
3. Push — the pipeline runs automatically

```bash
# Example: add the Python template to your repo
cp templates/python/ci.yml /your-repo/.github/workflows/ci.yml
```

---

## Common features (all templates)

- **Node.js 24** — all actions use `v6`+ versions
- **Deploy only on `main`** — PRs get lint/test/build but never deploy
- **`workflow_dispatch`** — manual trigger with configurable inputs
- **Semver tagging** — Docker images tagged by git tag, branch, or SHA
- **GHA cache** — Docker layer cache via `type=gha` for fast rebuilds
- **Security scans** — Trivy, pip-audit/npm-audit, Gitleaks, Semgrep, Checkov
- **Artifacts** — reports uploaded and retained for 7–30 days

---

## Deploy targets

All templates support two deploy targets:

| Target | How |
|--------|-----|
| **GHCR** | Push Docker image to `ghcr.io/<owner>/<app>` |
| **VPS SSH** | SSH into server, pull image, restart via `docker compose` |

See [docs/deploy-targets.md](docs/deploy-targets.md) for setup instructions.

---

## Required secrets

See [docs/secrets-setup.md](docs/secrets-setup.md) for the full list per template.

Quick reference — secrets common to all deploy templates:

| Secret | Description |
|--------|-------------|
| `GHCR_TOKEN` | GitHub token with `packages:write` scope |
| `VPS_HOST` | VPS IP or domain |
| `VPS_USER` | SSH user on VPS |
| `VPS_SSH_KEY` | Private SSH key (PEM format) |
| `VPS_PORT` | SSH port (default `22`) |

---

## Project structure

```
cicd-templates/
├── templates/
│   ├── python/ci.yml
│   ├── nodejs/ci.yml
│   ├── generic/ci.yml
│   ├── docker-only/ci.yml
│   ├── ansible/ci.yml
│   ├── terraform/ci.yml
│   └── security/ci.yml
├── docs/
│   ├── how-to-use.md
│   ├── secrets-setup.md
│   └── deploy-targets.md
└── .github/workflows/
    └── validate.yml       # Validates all template YAMLs on every push
```

---

## Changelog

### v1.0.3
- feat: matrix builds in `python` template (3.11/3.12/3.13) with coverage artifact per version — closes #5
- feat: matrix builds in `nodejs` template (Node 20/22/24) with coverage artifact per version — closes #5
- feat: matrix example (commented) added to `generic` template — closes #5
- feat: `python/release.yml` — semantic release via `python-semantic-release` — closes #6
- feat: `nodejs/release.yml` — semantic release via `npx semantic-release` — closes #6
- feat: `generic/release.yml` — stack-agnostic release with built-in version calculator — closes #6

### v1.0.2
- feat: post-deploy health check in python, nodejs, docker-only templates — closes #3
  (polls `/health` every 5s for up to 60s; fails pipeline on timeout)
- docs: OIDC authentication using `GITHUB_TOKEN` — closes #2
  (no long-lived `GHCR_TOKEN` secret needed; `permissions: packages: write`)

### v1.0.1
- feat: `timeout-minutes` on every job in all 7 templates — closes #1
- feat: `concurrency` group cancels in-progress runs on same branch — closes #4
- chore: Dependabot for GitHub Actions (weekly)

---

## License

MIT
