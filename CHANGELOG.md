# Changelog

All notable changes to this project are documented here. See the
[README](README.md) for current features and usage.

### v1.7.0
- feat: **real GitHub Releases for this repo**
  (`.github/workflows/release.yml`) — pushing a `vX.Y.Z` tag now creates a
  GitHub Release whose body is exactly that version's `CHANGELOG.md`
  section, not the whole file. `CHANGELOG.md` keeps the full running
  history; each Release only shows what changed in that version.
- fix: **`dependency-review-action`'s `continue-on-error` removed** — was
  added temporarily after the check hard-errored on first run because
  "Dependency graph" wasn't enabled for this repo; now confirmed on, so
  the check is blocking again as originally intended.

### v1.6.0
- feat: **`dependency-review-action` on this repo's own PRs**
  (`.github/workflows/dependency-review.yml`) — blocks merging a PR that
  introduces a dependency with a known vulnerability. Same scope caveat as
  Dependabot: only covers `.github/workflows/`, since GitHub's dependency
  graph doesn't see the actions pinned inside `templates/*.yml`.
- feat: **OpenSSF Scorecard for this repo itself**
  (`.github/workflows/scorecard.yml`) — distinct from the existing
  `ossf/scorecard-action` step inside the `nodejs` template (which scores
  whatever repo *adopts* that template); this one scores `cicd-templates`
  and publishes a badge in the README.
- docs: added `ROADMAP.md`.

### v1.5.0
- fix(security): **all third-party actions pinned to commit SHA, not a
  mutable tag** — closes a supply-chain gap flagged by a cross-repo
  security review (#20): a compromised or malicious upstream maintainer
  could otherwise re-point an existing tag to different content without
  this repo's workflow files changing at all. Every `uses:` line across
  `templates/` and `.github/workflows/` now pins the exact commit SHA
  (resolved via `git ls-remote` against the real upstream repo), with the
  human-readable tag kept as a trailing comment. Also re-closed the
  `templates/` vs `.github/workflows/` version drift that had crept back
  in since v1.3.0 (`actions/setup-node`, `aquasecurity/trivy-action`,
  `docker/setup-qemu-action`, `hadolint/hadolint-action`,
  `ossf/scorecard-action`) before pinning, so there's one canonical
  version per action.
- fix(security): **`step-security/harden-runner` added as the first step
  of every job** (#20), across every template, reusable workflow,
  `validate.yml`, and `dependabot-auto-merge.yml`. Ships in
  `egress-policy: audit` (logs, never blocks) since these are generic
  templates with per-consumer dependency needs — tighten to `block` with
  an explicit `allowed-endpoints` list once you know your job's real
  network footprint.

### v1.4.0
- feat: **reusable `workflow_call` versions of `docker-only` and `generic`** —
  `.github/workflows/docker-only-ci-reusable.yml` and `generic-ci-reusable.yml`.
  `docker-only`'s adds a `deploy_target` input so callers can opt out of the
  VPS deploy job (the copy-paste template always runs it; a caller can't
  delete a job). `generic`'s exposes `setup_command`/`test_command` shell
  inputs in place of the copy-paste template's editable placeholder step,
  since a reusable workflow can't have a caller inject arbitrary `uses:`
  steps — documented in the file, with a pointer back to copying the
  template for stacks that need a pinned toolchain action.
- feat: **SLSA provenance attestation** — `actions/attest-build-provenance@v4`
  runs alongside the existing SBOM + cosign signing in `python`, `nodejs`,
  `generic`, and `docker-only` (templates and reusable workflows), recording
  which workflow/commit/inputs produced the image. README documents
  `gh attestation verify` alongside the existing `cosign verify` example.
- feat: **Dependabot auto-merge** — `.github/workflows/dependabot-auto-merge.yml`
  auto-merges this repo's own Dependabot PRs once CI passes, but only for
  minor/patch bumps; major bumps get a comment asking for a human review
  instead. Uses GitHub's documented `dependabot/fetch-metadata` +
  `gh pr merge --auto` pattern — doesn't bypass CI, needs "Allow auto-merge"
  enabled in repo settings.
- feat: **multi-version test matrix for `python` and `nodejs`** — lint/test
  now run as a `strategy.matrix`. Default is a single-entry matrix (identical
  cost to before); add entries, or set the reusable workflows'
  `python_versions`/`node_versions` input, to test multiple runtime versions.

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
