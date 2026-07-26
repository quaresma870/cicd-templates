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
- [ ] `docs/branch-protection.md` — guide for consumers on required status
      checks / branch protection for `main`, since every template assumes
      deploy-only-on-`main` semantics.
- [ ] CodeQL on this repo's own shell scripts / workflow files (distinct
      from `templates/security/ci.yml`, which scans *consumers'* repos).

## 2. New language templates

- [ ] **Go** — natural fit for this repo's Docker-centric model: static
      binary, trivial multi-stage Dockerfile, fast builds. Also the de
      facto language of cloud-native tooling.
- [ ] **Java** (Spring Boot, Gradle or Maven) — still dominant in
      enterprise/banking backends; biggest gap in current stack coverage.
- [ ] **TypeScript-first update to `templates/nodejs/`** — not a new
      stack, but TypeScript is now the most-used language on GitHub by
      contributors (Octoverse 2025). Add a `tsc --noEmit` type-check step
      to the lint stage and default examples to `.ts`.
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

- [ ] **Fly.io** — simplest to configure (no cluster to manage).
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
