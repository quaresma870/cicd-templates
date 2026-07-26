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
- [ ] **Rust** — smaller current usage but the most-admired language on
      Stack Overflow for years running and moving into real production
      in 2026; minimal container images, good fit for CLIs/high-perf
      services.
- [ ] **.NET / C#** — large enterprise/Azure footprint; lowest priority
      unless a consumer's stack actually needs it.

## 3. Additional deploy targets

Currently every template supports `ghcr` and `vps` (SSH + docker compose).
Candidates to add as new `deploy_target` values, across the relevant
templates and reusable workflows:

- [x] **Fly.io** — simplest to configure (no cluster to manage). Ships as
      a `deploy-fly` job across all 5 templates + 6 reusable workflows,
      using `flyctl deploy --remote-only` (builds from the Dockerfile on
      Fly's own remote builder, not the GHCR-pushed image — Fly.io has no
      clean native way to auth to a *private* GHCR image at deploy time).
- [ ] **Kubernetes** (kubectl apply / Helm upgrade) — most requested /
      most generic of the three; needs a kubeconfig secret convention.
- [ ] **AWS ECS** — needs a decision on OIDC federation vs. static AWS
      access keys before implementation.

## 4. Testing infrastructure

- [ ] `act`-based end-to-end template testing — actually runs a template's
      jobs (not just lints/semantically-validates them) inside CI. Can't
      be verified locally in every environment (needs working
      Docker-in-Docker); would need to be validated against a real GitHub
      Actions run and iterated from there if it fails.
