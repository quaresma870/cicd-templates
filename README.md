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

`python/`, `nodejs/`, `generic/`, `docker-only/`, `ansible/`, and `terraform/`
each include a `release.yml` for automated semantic-version tagging from
conventional commits. **`security/` deliberately doesn't** — it's a scanning
pipeline you bolt onto an existing repo, not a deployable artifact with its
own version number; there's nothing meaningful for a release workflow to tag
or publish. If you do want to track when the scanning configuration itself
changed, a plain git tag on this repo (not the one you're scanning) is enough
— no dedicated workflow needed for that.

---

## How to use

Two ways to use these templates — pick whichever fits:

### Option A — Copy (full control)

1. Copy the relevant `ci.yml` to your repo's `.github/workflows/` directory
2. Configure the required secrets and variables (see each template's README or [docs/secrets-setup.md](docs/secrets-setup.md))
3. Push — the pipeline runs automatically

```bash
# Example: add the Python template to your repo
cp templates/python/ci.yml /your-repo/.github/workflows/ci.yml
```

The workflow is now entirely yours — edit it freely. A fix or improvement
landing in this repo later **will not** reach you automatically; you'd need
to re-copy and re-diff by hand.

### Option B — Call (stays in sync, less editable)

Available for `python` so far ([`.github/workflows/python-ci-reusable.yml`](.github/workflows/python-ci-reusable.yml)
— GitHub requires reusable workflows to live directly in `.github/workflows/`,
not under `templates/`, so this is the one exception to this repo's usual layout).

```yaml
# .github/workflows/ci.yml in YOUR repo
name: CI

on: [push, pull_request]

permissions:
  contents: read
  id-token: write    # required — see the comment at the top of python-ci-reusable.yml for why
  packages: write     # only if deploy_target includes 'ghcr'

jobs:
  ci:
    uses: quaresma870/cicd-templates/.github/workflows/python-ci-reusable.yml@main
    with:
      python_version: "3.12"
      image_name: my-app
      deploy_target: ghcr     # ghcr | vps | both | none
    secrets:
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
      # VPS_HOST / VPS_USER / VPS_SSH_KEY / VPS_PORT — only if deploy_target includes vps
```

A fix or new best practice landing in `python-ci-reusable.yml` reaches every
caller automatically next run — at the cost of not being able to tweak a
single step without forking this repo. Pick **Option A** if you want full
ownership of the workflow file; pick **Option B** if you'd rather not
maintain your own copy and are fine with this repo's defaults.

---

## Common features (all templates)

- **Node.js 24** — all actions use `v6`+ versions
- **Deploy only on `main`** — PRs get lint/test/build but never deploy
- **`workflow_dispatch`** — manual trigger with configurable inputs
- **Semver tagging** — Docker images tagged by git tag, branch, or SHA
- **GHA cache** — Docker layer cache via `type=gha` for fast rebuilds
- **Security scans** — Trivy, pip-audit/npm-audit, Gitleaks, Semgrep, Checkov
- **Artifacts** — reports uploaded and retained for 7–30 days

`python/`, `nodejs/`, `generic/`, and `docker-only/` additionally generate a
software bill of materials and sign the built image on every push to `main`
(skipped on PR builds, which don't push an image):

- **SBOM** — [Syft](https://github.com/anchore/syft) via `anchore/sbom-action`, SPDX format, uploaded as a workflow artifact
- **Image signing** — [Cosign](https://github.com/sigstore/cosign) keyless signing via GitHub's OIDC token (no signing key to manage or rotate)

Verify a signed image before pulling it in production:

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/YOUR_ORG/YOUR_REPO/.github/workflows/.*@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/YOUR_ORG/YOUR_IMAGE:latest
```

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
│   ├── python/{ci,release}.yml
│   ├── nodejs/{ci,release}.yml
│   ├── generic/{ci,release}.yml
│   ├── docker-only/{ci,release}.yml
│   ├── ansible/{ci,release}.yml
│   ├── terraform/{ci,release}.yml
│   └── security/ci.yml          # no release.yml — see "Templates" above for why
├── docs/
│   ├── how-to-use.md
│   ├── secrets-setup.md
│   └── deploy-targets.md
└── .github/workflows/
    ├── validate.yml                # Validates all template YAMLs on every push
    └── python-ci-reusable.yml      # Callable alternative to templates/python/ci.yml — see "How to use"
```

---

See [CHANGELOG.md](CHANGELOG.md) for release history.

---

## License

MIT
