# Changelog

All notable changes to this project are documented here. See the
[README](README.md) for current features and usage.

### v1.20.0
- feat: **.NET template** — `templates/dotnet/ci.yml`, `templates/dotnet/release.yml`,
  and `.github/workflows/dotnet-ci-reusable.yml` shipped, closing out
  `ROADMAP.md`'s "New language templates" section. Same Lint → Test →
  Security → Build → Deploy shape as every other language template, with
  all four current deploy targets (`ghcr`/`vps`/`fly`/`k8s`/`ecs`).
  - **Lint:** `dotnet format --verify-no-changes` — part of the SDK
    itself since .NET 6, no extra tool install needed.
  - **Test:** `dotnet test --collect:"XPlat Code Coverage"` (via the
    `coverlet.collector` package every `dotnet new xunit`/`nunit`/`mstest`
    project already references by default), then the same
    parse-Cobertura-XML-and-enforce-threshold approach as Java's
    `jacoco.xml` handling — 70% default line coverage.
  - **Security:** `dotnet list package --vulnerable --include-transitive`
    — built into the SDK since .NET 5, no extra tool needed; wrapped with
    a grep-based check since the command itself doesn't fail the build
    on findings.
  - **Build/Deploy:** identical Docker + SBOM + cosign + SLSA attestation
    + VPS/Fly.io/Kubernetes/AWS ECS deploy pattern as every other
    Docker-building template.
  - `release.yml` bumps every `.csproj`'s `<Version>` element to match
    the computed tag — a .NET project's version genuinely lives there,
    same reasoning as Java's `pom.xml` / Rust's `Cargo.toml` bump.
    Verified the `sed` logic against a real `.csproj` locally before
    shipping. Assumes the common "one main project" layout; a solution
    with multiple independently-versioned projects needs a different
    approach.
  - Defaults to .NET 10 (current LTS, released November 2025) via
    `actions/setup-dotnet`.
  - README, `docs/secrets-setup.md`, and `ROADMAP.md` updated.

### v1.19.0
- feat: **Rust template** — `templates/rust/ci.yml`, `templates/rust/release.yml`,
  and `.github/workflows/rust-ci-reusable.yml` shipped, using the same
  Lint → Test → Security → Build → Deploy shape as every other language
  template, with all four current deploy targets (`ghcr`/`vps`/`fly`/`k8s`/`ecs`).
  - **Lint:** `cargo fmt --all -- --check` + `cargo clippy --all-targets
    --all-features -- -D warnings`.
  - **Test:** `cargo llvm-cov` (source-based coverage via the
    `llvm-tools-preview` rustup component) enforces the same 70% default
    line-coverage threshold as `python`/`nodejs`/`go`/`java`, in one
    command (`--fail-under-lines`) rather than a separate parse-and-check
    step.
  - **Security:** `cargo audit` — the official Rust security-advisory
    scanner.
  - **Build/Deploy:** identical Docker + SBOM + cosign + SLSA attestation
    + VPS/Fly.io/Kubernetes/AWS ECS deploy pattern as every other
    Docker-building template.
  - `release.yml` bumps `Cargo.toml`'s `version` (and, if present,
    `Cargo.lock`'s matching entry) to match the computed tag — a crate's
    version genuinely lives in `Cargo.toml`, same reasoning as Java's
    `pom.xml` bump. Uses a plain `sed` substitution rather than
    `cargo set-version` (from the `cargo-edit` crate, not part of a
    stock Rust install) to avoid an extra from-source compile in CI.
  - Uses `taiki-e/install-action` to install `cargo-llvm-cov`/`cargo-audit`
    as prebuilt binaries rather than `cargo install` (which compiles from
    source — several minutes per tool). `dtolnay/rust-toolchain` is
    pinned to its `stable` branch's current commit rather than a release
    tag — that action has no version tags at all, `@stable` is itself a
    branch name selecting the toolchain channel; the pin only protects
    the action's own code, `rustup` still resolves the actual current
    stable Rust release at runtime.
  - `Swatinem/rust-cache`'s SHA confirmed against its peeled `^{}` ref
    (an annotated tag) before pinning.
  - README, `docs/secrets-setup.md`, and `ROADMAP.md` updated.

### v1.18.0
- feat: **AWS ECS deploy target** — third and last item of `ROADMAP.md`'s
  "Additional deploy targets" section, completing it. All 5 copy-paste
  templates (`python`, `nodejs`, `go`, `java`, `generic`) and all 6
  reusable workflows gain an `ecs` option alongside
  `ghcr | vps | fly | k8s | both | none`, plus a new `deploy-ecs` job.
  Uses **static AWS access keys** (`AWS_ACCESS_KEY_ID` /
  `AWS_SECRET_ACCESS_KEY`), not OIDC federation — a deliberate choice, not
  a placeholder.
  `deploy-ecs` deliberately only rolls out a new image onto an ECS
  service + task definition family that must already exist: `aws ecs
  describe-task-definition` → `aws-actions/amazon-ecs-render-task-definition`
  patches in the new image → `aws-actions/amazon-ecs-deploy-task-definition`
  registers the new revision, updates the service, and waits for
  stability. Same scope limit as `k8s` — doesn't create the cluster,
  service, or task definition, since that's environment-specific and out
  of scope.
  New `AWS_REGION`/`ECS_CLUSTER` variables (required, no sensible
  default) and `ECS_SERVICE` (defaults to `APP_NAME`/`image_name`).
  Documented the `repositoryCredentials` prerequisite for task
  definitions pulling from a private GHCR repo (the default here) — this
  repo's own build job doesn't touch that field, only the image URI.
  Both `aws-actions/amazon-ecs-render-task-definition` and
  `aws-actions/amazon-ecs-deploy-task-definition` are annotated tags —
  confirmed against their peeled `^{}` refs before pinning.
  `docs/secrets-setup.md` and `docs/deploy-targets.md` updated. README
  and `ROADMAP.md` updated.

### v1.17.0
- fix(security): **OpenSSF Scorecard remediation for this repo itself**
  (score was 4.8/10 as of the last scan). Fixed what's actually fixable
  via code:
  - `validate.yml` had no `permissions:` block at all — added
    `contents: read` at the workflow level.
  - `dependabot-auto-merge.yml` and `release.yml` granted `contents:
    write` (plus `pull-requests: write` for the former) at the
    *workflow* level — Scorecard's Token-Permissions check specifically
    penalizes that even when a workflow has only one job, since it's not
    scoped against future jobs being added. Moved both to job-level
    permissions with a `contents: read` workflow-level default.
  - Added `LICENSE` (MIT) — the README already claimed an MIT badge, but
    no actual license file existed.
  - Added `SECURITY.md` — points reporters to GitHub's private
    vulnerability reporting instead of a public issue.
  - Not fixable via code, left as-is: `Maintained` (repo age; ages out
    naturally), `Contributors` (single-maintainer repo), `Fuzzing` /
    `Packaging` (not applicable to a YAML template collection),
    `CII-Best-Practices` (requires manually registering on
    bestpractices.dev — a human action, not something a commit can do),
    `Branch-Protection` and `Code-Review` (real repository settings
    changes, not committed code — see `docs/branch-protection.md`, which
    already documents the recommended configuration for repos that
    *adopt* a template; this repo hasn't applied that same guidance to
    itself yet, and doing so is a deliberate decision left to the repo
    owner rather than something to silently flip on).

### v1.16.0
- feat: **Kubernetes deploy target** — second item of `ROADMAP.md`'s
  "Additional deploy targets" section. Same reach as Fly.io: all 5
  copy-paste templates (`python`, `nodejs`, `go`, `java`, `generic`) and
  all 6 reusable workflows gain a `k8s` option alongside
  `ghcr | vps | fly | both | none`, plus a new `deploy-k8s` job.
  `deploy-k8s` deliberately only rolls out a new image tag — it runs
  `kubectl set image deployment/<app> <app>=<new-image> -n <namespace>`
  then waits on `kubectl rollout status`. It doesn't apply manifests or
  create resources; that's cluster/environment-specific (Helm, raw
  manifests, Kustomize, GitOps) and out of scope, same way `vps` assumes
  `docker-compose.yml` is already on the server rather than the pipeline
  creating it. Kubeconfig comes in as a base64-encoded `KUBE_CONFIG`
  secret, decoded to `~/.kube/config` on the runner and deleted
  afterward regardless of outcome. New `KUBE_NAMESPACE` variable
  (`kube_namespace` input for reusable workflows) defaults to `default`.
  `azure/setup-kubectl`'s SHA confirmed against its peeled `^{}` ref
  (v5 is an annotated tag) before pinning.
  `docs/secrets-setup.md` and `docs/deploy-targets.md` document the new
  secret/variable and the "Deployment must already exist" prerequisite.
  README and `ROADMAP.md` updated.

### v1.15.0
- feat: **Fly.io deploy target** — first item of `ROADMAP.md`'s
  "Additional deploy targets" section. All 5 copy-paste templates
  (`python`, `nodejs`, `go`, `java`, `generic`) and all 6 reusable
  workflows (adding `docker-only-ci-reusable.yml`) gain a `fly` option
  alongside `ghcr | vps | both | none`, plus a new `deploy-fly` job.
  Unlike `ghcr`/`vps`, `deploy-fly` doesn't reuse the image `build`
  already pushed to GHCR — it runs `flyctl deploy --remote-only`, which
  builds straight from the repo's `Dockerfile` on Fly's own remote
  builder. This is a deliberate tradeoff: Fly.io has no clean native way
  to authenticate to a *private* GHCR image at deploy time (a
  long-standing gap per Fly's own community forum), so building directly
  sidesteps that friction, at the cost of building the image twice.
  `fly` is always selected on its own — it isn't folded into
  `deploy_target: both` the way `ghcr`+`vps` are.
  `superfly/flyctl-actions/setup-flyctl`'s SHA confirmed as a lightweight
  tag (no `^{}` peeling needed) before pinning.
  `docs/secrets-setup.md` and `docs/deploy-targets.md` document the new
  `FLY_API_TOKEN` secret and the `fly.toml` prerequisite (`fly launch`).
  README and `ROADMAP.md` updated.

### v1.14.0
- feat: **TypeScript-first update to the Node.js template** — TypeScript
  is now the most-used language on GitHub by contributors (Octoverse
  2025), so `templates/nodejs/ci.yml`'s lint job now runs `tsc --noEmit`
  alongside your own `npm run lint` (assumes a `tsconfig.json` in the
  repo root — delete the one step for a plain JavaScript project).
  `nodejs-ci-reusable.yml` exposes this as a `type_check` input
  (default `true`, set `false` to skip) since a reusable-workflow caller
  can't delete a step the way a copy-paste template user can. README
  templates table, Option B docs, and `ROADMAP.md` updated. This closes
  out `ROADMAP.md`'s "New language templates" section except Rust and
  .NET/C#, both explicitly lower priority.

### v1.13.0
- feat: **Java template (Maven)** — `templates/java/ci.yml`,
  `templates/java/release.yml`, and the reusable `java-ci-reusable.yml`,
  completing the same Lint → Test → Security → Build → Deploy shape every
  other language template follows.
  - **Lint:** Checkstyle (`mvn checkstyle:check`, Sun's default ruleset
    unless `pom.xml` overrides it).
  - **Test:** JaCoCo coverage, invoked via direct plugin-goal coordinates
    (`prepare-agent` + `test` + `report`) so it works without requiring
    `pom.xml` to already declare the plugin; a small Python step parses
    `jacoco.xml` for the line-coverage percentage and enforces the same
    70% default threshold as `python`/`nodejs`/`go`.
  - **Security:** OWASP Dependency-Check
    (`mvn org.owasp:dependency-check-maven:check`) — also invoked via
    full coordinates, no `pom.xml` config required.
  - **Build/Deploy:** identical Docker + SBOM + cosign + SLSA attestation
    + VPS-deploy pattern as every other build-a-Docker-image template.
  - `release.yml` bumps `pom.xml`'s `<version>` to match the computed
    tag before creating the Release — unlike Go, a Maven artifact's
    version genuinely lives in the POM, not just the tag.
  - Defaults to Java 25 (current LTS, released September 2025) via
    Eclipse Temurin.
- chore: re-verified every SHA pin in the repo (33 total, including this
  template's 2 new `actions/setup-java` references) against `git
  ls-remote`'s explicit peeled `^{}` ref — the query shape that v1.12.0
  and v1.12.1 each had to fix in turn. Zero mismatches this time.
- README, `docs/secrets-setup.md`, and `ROADMAP.md` updated.

### v1.12.1
- fix(security): **`github/codeql-action` had the same annotated-tag-object
  bug as v1.12.0's 7 pins** — missed in that audit because the script's
  regex only matched two-segment `owner/repo@sha` paths, and
  `github/codeql-action/init`, `.../analyze`, and `.../upload-sarif` are
  three-segment (`owner/repo/subpath`) action references. Confirmed the
  hard way again: `scorecard.yml` kept failing on `main` after v1.12.0
  merged, this time rejecting `github/codeql-action/upload-sarif`'s pin
  as an imposter commit. Re-ran the audit with a regex that correctly
  splits the leading `owner/repo` from any subpath before resolving —
  every SHA pin in the repo (`codeql.yml`'s two uses plus
  `scorecard.yml`'s one) now verified against the real peeled commit SHA,
  zero mismatches.

### v1.12.0
- fix(security): **7 SHA pins were actually the annotated-tag object SHA,
  not the real commit SHA** — `actions/attest-build-provenance`,
  `aquasecurity/trivy-action`, `aws-actions/configure-aws-credentials`,
  `gitleaks/gitleaks-action`, `golangci/golangci-lint-action`,
  `ossf/scorecard-action`, and `sigstore/cosign-installer`. Root cause:
  the original SHA-resolution script only checked the peeled `^{}` ref as
  a fallback when the plain `refs/tags/<tag>` query came back empty, but
  an annotated tag's plain query is never empty — it returns the tag
  *object's* SHA, so the fallback never ran. Confirmed on real CI: this
  repo's own `scorecard.yml` started failing with `ossf/scorecard-action`
  rejecting its own pin as an "imposter commit" (its backend verifies the
  pinned SHA against the repo's actual commit history, which a tag-object
  SHA fails). Audited and fixed all 31 SHA pins across the repo — the
  other 24 were already correct. `CONTRIBUTING.md`'s SHA-pinning guidance
  now explicitly calls out checking for the `^{}` line.

### v1.11.0
- feat: **Go template** — `templates/go/ci.yml`, `templates/go/release.yml`,
  and the reusable `go-ci-reusable.yml`, completing the same
  Lint → Test → Security → Build → Deploy shape every other language
  template follows. Lint uses `golangci-lint` (v2.12 via
  `golangci-lint-action`), security scan uses `govulncheck` — the official
  Go vulnerability scanner from the `golang` org, checked against actual
  call graphs rather than just declared dependencies. Test enforces a
  coverage threshold computed from `go tool cover`, same 70% default as
  the other templates. `release.yml` is tag-based like `ansible`'s, but
  for a more fundamental reason: Go modules are versioned directly by git
  tag (`go get module@v1.2.3` resolves straight to it), so the tag +
  Release this workflow creates *is* the actual release artifact, not
  just a changelog convenience — includes a commented-out reminder for
  the `/v2`+ import-path convention Go modules require for major
  versions. README, `docs/secrets-setup.md` (identical secrets to
  `python/`), and `ROADMAP.md` updated.

### v1.10.0
- feat: **CodeQL for this repo itself** (`.github/workflows/codeql.yml`) —
  uses CodeQL's `actions` language (GA since April 2025), a purpose-built
  static analysis for GitHub Actions workflow files: script injection via
  untrusted inputs in `run:` steps, missing/overly-broad permissions,
  dangerous `pull_request_target` use, etc. A direct match for what this
  repo actually contains, so this replaced the more generic "scan shell
  scripts" idea from `ROADMAP.md` once researched. Distinct from
  `templates/security/ci.yml`, which scans *consumers'* repos, not this
  one. README badge added; project-structure listing in the README
  brought back in sync with the actual `.github/workflows/` contents
  (`dependency-review.yml`, `scorecard.yml`, `release.yml` were missing
  from it). This closes out `ROADMAP.md`'s "Repo process & supply-chain
  polish" section — next up is new language templates.

### v1.9.0
- docs: **`docs/branch-protection.md`** — guide for repos that adopt a
  template on which status checks to require (and which to deliberately
  *not* require, like VPS deploy jobs that only run on `main`), since
  every template's "deploy only on `main`" assumption is only actually
  enforced if `main` is protected. Covers required-PR-review, Code Owners
  review, the `dependabot-auto-merge.yml` + branch-protection interaction,
  and optional tag protection for the new `release.yml` pattern.

### v1.8.0
- feat: **`python-package-ci-reusable.yml`** — a new, deliberately leaner
  reusable workflow for installable Python CLI/library packages (built as a
  wheel/sdist, no Dockerfile, no registry push, no VPS deploy). The existing
  four reusable workflows (`python`, `nodejs`, `docker-only`, `generic`) all
  assume a "build Docker image → push to GHCR → deploy to VPS" shape, which
  is a real, confirmed mismatch for a portfolio of CLI tools that are never
  containerized or deployed anywhere at all. This one covers only the
  genuinely identical part across such a portfolio — lint, build, verify
  metadata, smoke-install in a clean venv — and deliberately does not run
  the calling package's own test suite or any bespoke integration testing,
  since those are usually the most valuable, most repo-specific part of a
  given package's CI and don't generalize into a shared template without
  losing what makes them useful. See the README's "How to use" Option C.
  First validated end-to-end against a real caller (`voipaudit`) before
  being adopted more broadly.

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
