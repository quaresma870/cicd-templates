# Contributing

Thanks for considering a contribution to these CI/CD templates.

## Repo layout

- `templates/<stack>/ci.yml` — copy-paste templates. Self-contained, no
  dependency on anything else in this repo.
- `templates/<stack>/release.yml` — optional companion, tag-based semantic
  release triggered by `workflow_run` after `ci.yml` succeeds on `main`.
- `.github/workflows/*-reusable.yml` — `workflow_call` alternative to a
  copy-paste template (currently only `python`). Must live directly in
  `.github/workflows/` — GitHub doesn't support reusable workflows under
  subdirectories.
- `.github/workflows/validate.yml` — CI for this repo itself: yamllint +
  actionlint against every template and reusable workflow on every push/PR.

## Adding or changing a template

1. Keep the stage structure consistent with the other templates (see the
   table in the README) unless the stack genuinely doesn't need a stage.
2. Pin third-party actions to the exact commit SHA, never a floating tag
   like `@main`, `@latest`, or even a version tag like `@v7` on its own —
   tags are mutable. Resolve the SHA for the tag you actually want with
   `git ls-remote <repo-url> refs/tags/<tag>` (don't guess or hallucinate a
   SHA), then pin it with the tag kept as a trailing comment for
   readability: `uses: owner/repo@<sha> # v7`. Dependabot understands this
   format and keeps the SHA current automatically.
3. Every job's first step should be `step-security/harden-runner` with
   `egress-policy: audit` (see any existing job for the exact block) —
   monitors network egress without blocking anything, so it's always safe
   to add.
4. If you touch a step that reads a secret inside `environment.url` or an
   `if:` condition, remember `secrets` is **not** available in those two
   contexts — route it through a step output or an `env:` var instead (see
   the existing `Compute environment URL` steps for the pattern). This has
   been a real, shipped bug here before (see CHANGELOG v1.2.0).
5. Run yamllint and actionlint locally before opening a PR:
   ```bash
   pip install yamllint
   yamllint -d '{extends: relaxed, rules: {line-length: {max: 150}}}' templates/<stack>/ci.yml
   docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:latest templates/<stack>/ci.yml
   ```
6. Update the README's template table and `docs/secrets-setup.md` if you add
   new required secrets or variables.

## Keeping action versions in sync

Dependabot only scans `.github/workflows/` — it cannot see or bump the
actions pinned inside `templates/*.yml`, since those are example files this
repo never actually executes. When a Dependabot PR bumps an action here
(e.g. `actions/checkout`), check whether the same action is pinned in
`templates/*/ci.yml` or `templates/*/release.yml` and bump it there too in
the same PR, so the copy-paste templates don't silently drift from the
versions this repo has already validated in its own CI.

## Adding a new reusable (`workflow_call`) workflow

`python-ci-reusable.yml` is the reference implementation. To add one for
another stack:

1. Base it on the matching `templates/<stack>/ci.yml`, converting hardcoded
   values (`vars.APP_NAME`, `env.PYTHON_VERSION`, etc.) into `workflow_call`
   `inputs:`.
2. Mirror the `secrets:` block — mark `GHCR_TOKEN` required, VPS secrets
   optional.
3. Document the exact `permissions:` block a caller needs (`id-token: write`
   is required for cosign signing — a caller missing it fails at dispatch
   time, before any job runs).
4. Add it to `validate.yml`'s reusable-workflow lint pass (matches
   `*-reusable.yml` automatically — no change needed there) and to the
   README's "How to use" section.

## Commit messages

This repo tags releases from conventional commits (`feat:`, `fix:`,
`docs:`, `chore:`, `build:` …) — use that prefix convention so `release.yml`
computes the right version bump for templates that consume it.
