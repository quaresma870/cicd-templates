# Changelog

All notable changes to this project are documented here. See the
[README](README.md) for current features and usage.

### v1.3.0
- feat: **reusable `workflow_call` version of the Node.js template** —
  `.github/workflows/nodejs-ci-reusable.yml`, same tradeoffs and usage
  pattern as `python-ci-reusable.yml` (see README "How to use").
- fix: **templates/ action versions had drifted from `.github/workflows/`** —
  Dependabot only scans `.github/workflows/`, so its recent bumps
  (`actions/checkout` v6→v7, `actions/setup-python` v6→v7,
  `actions/upload-artifact` v4→v7, `appleboy/ssh-action` v1.2.0→v1.2.5)
  never reached the copy-paste templates. Synced all `templates/*.yml` and
  `templates/*/release.yml` to match, and documented the gap in
  `CONTRIBUTING.md` so it's checked by hand going forward.
- chore: **Dependabot groups all `github-actions` bumps into a single weekly
  PR** instead of one PR per dependency.
- docs: added `CONTRIBUTING.md` and `.github/CODEOWNERS`.

### v1.2.0
- fix: **6 real bugs — secrets referenced where the context isn't available** — 5 templates
  (python, docker-only, generic, nodejs, ansible) referenced `${{ secrets.VPS_HOST }}` (or
  `INVENTORY_HOST`) directly inside `jobs.<job_id>.environment.url`, and `security/ci.yml`
  referenced `secrets.SLACK_WEBHOOK` directly inside a `steps.if` condition — both genuinely
  disallowed per GitHub's own Contexts reference. Confirmed against the current, official
  documentation before fixing (not a stale-linter false positive), and fixed using GitHub's own
  documented patterns (a step output for the environment URL case; the same `env:` indirection
  workaround already used in the sibling infra-as-code repo for the `if:` case).
- feat: **new `actionlint` semantic-check step in `validate.yml`** — `yamllint` alone only checks
  YAML *syntax*, with no concept of what a valid *workflow* actually looks like, which is exactly
  the class of gap that let the 6 bugs above ship undetected. Verified to catch a real regression
  by deliberately reintroducing one of the bugs and confirming a precise, actionable annotation.
- fix: 2 additional real (if minor) shellcheck findings — `read` without `-r` and the
  `A && B || C` anti-pattern — surfaced by actionlint's own shellcheck integration once it was
  wired up, in `validate.yml`'s own scripts and one template.

### v1.1.0
- feat: **`release.yml` added to `docker-only`, `ansible`, and `terraform`** — same tag-based,
  conventional-commit-driven version calculation as `generic/release.yml`'s pattern (no
  `package.json`/`pyproject.toml` to bump for these stacks). `docker-only` additionally re-tags the
  already-built `:latest` image with the computed semantic version. `terraform` re-runs
  `terraform validate` against the exact tagged commit before tagging — belt-and-suspenders given
  how directly other infrastructure may pin to that tag via a git-ref module source.
  `templates/security/` deliberately does **not** get one — documented explicitly why (it's a
  scanning pipeline bolted onto an existing repo, not a deployable artifact with its own version).
- feat: **SBOM generation + keyless container image signing** — `python`, `nodejs`, `generic`, and
  `docker-only`'s build jobs now generate a software bill of materials
  ([Syft](https://github.com/anchore/syft) via `anchore/sbom-action`, SPDX format, uploaded as a
  90-day artifact) and sign the pushed image with [Cosign](https://github.com/sigstore/cosign) via
  keyless OIDC signing — no signing key to manage, rotate, or leak. README includes a
  `cosign verify` example for checking a signed image before pulling it in production.
- feat: **reusable workflow_call version of the Python template** — `.github/workflows/python-ci-reusable.yml`,
  an alternative to copying `templates/python/ci.yml` (not a replacement): call it from your own
  repo instead, and a fix landing here reaches you automatically. Must live directly in
  `.github/workflows/` rather than under `templates/` — a GitHub Actions platform requirement,
  confirmed against official docs rather than assumed. Found and fixed two genuinely non-obvious
  bugs by actually reproducing them on a throwaway test branch, not just by reading the YAML: (1)
  a calling workflow's `permissions:` are a hard ceiling on what the called workflow's jobs can be
  granted — omitting `id-token: write` on the *caller* breaks cosign signing with an opaque,
  job-less "workflow file issue" at dispatch time; (2) `environment.url` cannot reference the
  `secrets` context at all, which fails an entire run the same opaque way. Both are now documented
  prominently in the file itself, not just fixed silently. Verified end-to-end against a minimal
  real Python fixture: lint/test/security all passed for real, build correctly reached the actual
  `docker buildx build --push` step.
- chore: `validate.yml` extended to also yamllint any `*-reusable.yml` file under
  `.github/workflows/`, so the new file doesn't silently bypass the check that covers every other
  workflow file in this repo.

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
