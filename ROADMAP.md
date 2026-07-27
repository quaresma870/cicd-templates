# Roadmap

Tracks planned work beyond what's already shipped (see [CHANGELOG.md](CHANGELOG.md)
for what's done). Rough priority order within each section; sections
themselves aren't strictly sequential — pick whatever's most useful next.

## 1. Repo process & supply-chain polish

- [x] `actions/dependency-review-action` on this repo's own PRs — blocks
      introducing a dependency with a known vulnerability before merge.
- [x] OpenSSF Scorecard for this repo itself (not just the `nodejs`
      template) — publishes a security score badge for `cicd-templates`.
- [x] Dogfood a real release flow — on `vX.Y.Z` tag push, a workflow
      creates a GitHub Release whose body is exactly that version's
      `CHANGELOG.md` section (not the whole file). Closes the gap where
      versions exist only as CHANGELOG headers with no matching git tag
      or Release today.
- [x] `docs/branch-protection.md` — guide for consumers on required status
      checks / branch protection for `main`, since every template assumes
      deploy-only-on-`main` semantics.
- [x] CodeQL on this repo's own workflow files (distinct from
      `templates/security/ci.yml`, which scans *consumers'* repos) — uses
      CodeQL's purpose-built `actions` language (GA since April 2025)
      rather than a generic shell/YAML scan.

## 2. New language templates

- [x] **Go** — natural fit for this repo's Docker-centric model: static
      binary, trivial multi-stage Dockerfile, fast builds. Also the de
      facto language of cloud-native tooling. Both `templates/go/` and
      `go-ci-reusable.yml` shipped, using `golangci-lint` (lint) and
      `govulncheck` (the official Go vulnerability scanner, security scan).
- [x] **Java** (Spring Boot, Maven) — still dominant in enterprise/banking
      backends. `templates/java/` and `java-ci-reusable.yml` shipped:
      Checkstyle (lint), JaCoCo coverage threshold (test), OWASP
      Dependency-Check (security) — all invoked via direct Maven
      plugin-goal coordinates so they work without requiring the plugins
      to already be declared in a stock `pom.xml`. `release.yml` bumps
      `pom.xml`'s `<version>` to match the computed tag, since that's
      genuinely where a Maven artifact's version lives (unlike Go).
- [x] **TypeScript-first update to `templates/nodejs/`** — not a new
      stack, but TypeScript is now the most-used language on GitHub by
      contributors (Octoverse 2025). Lint job now runs `tsc --noEmit`
      alongside `npm run lint`; the reusable workflow exposes a
      `type_check` input (default `true`) since a plain-JS caller can't
      delete a job's step the way a copy-paste template user can.
- [x] **Rust** — smaller current usage but the most-admired language on
      Stack Overflow for years running and moving into real production
      in 2026; minimal container images, good fit for CLIs/high-perf
      services. `templates/rust/` and `rust-ci-reusable.yml` shipped:
      `cargo fmt`/`cargo clippy` (lint), `cargo-llvm-cov` coverage
      threshold (test), `cargo audit` (security) — all installed as
      prebuilt binaries via `taiki-e/install-action` rather than compiled
      from source. `release.yml` bumps `Cargo.toml`'s `version` to match
      the computed tag, since that's genuinely where a crate's version
      lives (unlike Go). All four deploy targets (`ghcr`/`vps`/`fly`/`k8s`/`ecs`)
      included from the start.
- [x] **.NET / C#** — large enterprise/Azure footprint. `templates/dotnet/`
      and `dotnet-ci-reusable.yml` shipped: `dotnet format` (lint),
      `dotnet test` + Cobertura XML coverage threshold (test),
      `dotnet list package --vulnerable` (security) — all built into the
      SDK itself since .NET 5/6, no extra tool install needed.
      `release.yml` bumps every `.csproj`'s `<Version>` to match the
      computed tag, since that's genuinely where a .NET project's version
      lives (unlike Go). Defaults to .NET 10 (current LTS). This closes
      out the "New language templates" section — Go, Java, TypeScript-
      first Node.js, Rust, and .NET are all shipped.

## 3. Additional deploy targets

Currently every template supports `ghcr` and `vps` (SSH + docker compose).
Candidates to add as new `deploy_target` values, across the relevant
templates and reusable workflows:

- [x] **Fly.io** — simplest to configure (no cluster to manage). Ships as
      a `deploy-fly` job across all 5 templates + 6 reusable workflows,
      using `flyctl deploy --remote-only` (builds from the Dockerfile on
      Fly's own remote builder, not the GHCR-pushed image — Fly.io has no
      clean native way to auth to a *private* GHCR image at deploy time).
- [x] **Kubernetes** — most requested / most generic of the three. Ships as
      a `deploy-k8s` job across all 5 templates + 6 reusable workflows,
      using `kubectl set image` + `kubectl rollout status` against a
      Deployment that must already exist in the cluster (no manifest
      apply/create — that's cluster-specific and out of scope). Kubeconfig
      convention: base64-encoded `KUBE_CONFIG` secret, decoded on the
      runner and deleted afterward.
- [x] **AWS ECS** — static AWS access keys (`AWS_ACCESS_KEY_ID` /
      `AWS_SECRET_ACCESS_KEY`), not OIDC federation. Ships as a
      `deploy-ecs` job across all 5 templates + 6 reusable workflows,
      registering a new task definition revision + updating the service
      against a cluster/service/task-definition family that must already
      exist (no cluster/service/task-def creation — same scope limit as
      `k8s`). This closes out the "Additional deploy targets" section.

## 4. Testing infrastructure

- [ ] `act`-based end-to-end template testing — actually runs a template's
      jobs (not just lints/semantically-validates them) inside CI. Can't
      be verified locally in every environment (needs working
      Docker-in-Docker); would need to be validated against a real GitHub
      Actions run and iterated from there if it fails.

## 5. OpenSSF Scorecard follow-up (this repo's own score)

Score was 4.8/10 as of the last scan. Fixed what's fixable via committed
code (see CHANGELOG v1.17.0): least-privilege `permissions:` blocks on
`validate.yml`/`dependabot-auto-merge.yml`/`release.yml`, `LICENSE`,
`SECURITY.md`. What's left needs a repo-owner decision or action, not a
PR:

- [ ] **Branch-Protection** (currently 0/10) — `main` isn't a protected
      branch. `docs/branch-protection.md` already documents the
      recommended settings for repos that *adopt* a template; applying
      that same guidance to `main` here is the fix, but it's a real repo
      setting (Settings → Branches), not something to flip on silently —
      especially since requiring PR-review-approval would change how
      PRs on this repo get merged going forward.
- [ ] **Code-Review** (currently 0/10, "0/22 approved changesets") — tied
      to the above: no PR has had a human/bot approval before merging.
      Requesting a Copilot review per PR going forward (or requiring
      review in branch protection) is the lever, once decided.
- Not actionable at all: **CII-Best-Practices** (requires the repo owner
  to personally register on bestpractices.dev — an external account
  action), **Maintained** (repo age, ages out on its own),
  **Contributors** (single-maintainer repo), **Fuzzing**/**Packaging**
  (don't apply to a YAML template collection).
