# Changelog

All notable changes to this project are documented here. See the
[README](README.md) for current features and usage.

### v1.0.4
- fix: **explicit least-privilege `permissions:` added to all 7 `ci.yml` templates** —
  none of them declared a `permissions:` block, meaning every job ran with whatever the
  repository/organisation's default `GITHUB_TOKEN` permissions happened to be, rather than
  exactly what each job actually needs. Confirmed none of the CI jobs need elevated access:
  container registry pushes use a separate `GHCR_TOKEN` secret (not `GITHUB_TOKEN`), so
  `contents: read` is sufficient at the workflow level for all of them. `templates/terraform/ci.yml`'s
  `plan` job is the one exception — it comments the plan output on pull requests via
  `actions/github-script`, so it gets a job-level override adding `pull-requests: write` on top
  of the same `contents: read` baseline. The `release.yml` templates already had this right
  (`contents: write` + `id-token: write`, scoped to the one job that needs it) — this brings the
  `ci.yml` templates up to the same standard, since these templates are copied directly into
  other people's real repositories and should model the practice, not just describe it.
- chore: removed leftover empty junk directories from an early shell command that didn't expand
  brace patterns as intended — never tracked in git, purely local clutter.
- verified (no fix needed): re-examined `validate.yml`'s `find templates/ -name "*.yml" | while
  read f; do ... exit 1; done` pattern for the classic bash gotcha where `exit` inside a piped
  `while` loop only exits the subshell, not the calling script. Tested empirically against this
  repo's actual GitHub Actions runner (a deliberately broken YAML file via a throwaway PR) rather
  than relying on general bash semantics — confirmed the workflow correctly fails as expected.
  No change made; documented here so the question doesn't get re-litigated from scratch later.

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
