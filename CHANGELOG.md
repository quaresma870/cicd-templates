# Changelog

All notable changes to this project are documented here. See the
[README](README.md) for current features and usage.

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
