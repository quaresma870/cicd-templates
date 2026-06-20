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

See [CHANGELOG.md](CHANGELOG.md) for release history.

---

## License

MIT
