# Security Policy

## Scope

This repo ships CI/CD pipeline templates and reusable GitHub Actions
workflows. In scope for a security report:

- A vulnerability in one of these templates/workflows themselves (e.g. a
  script-injection path via untrusted `${{ }}` interpolation, an action
  pinned to the wrong commit, overly broad token permissions).
- A vulnerability in this repo's own `.github/workflows/` (the CI that
  builds/tests/releases this repo, not consumers' repos).

Out of scope: vulnerabilities in third-party actions this repo references
(report those upstream, to the action's own repo) and issues in a
downstream repo that merely *adopted* a template (report those to that
repo's own maintainers).

## Reporting a vulnerability

Please use GitHub's private vulnerability reporting instead of opening a
public issue: go to the **Security** tab → **Report a vulnerability**, or
open one directly at
[github.com/quaresma870/cicd-templates/security/advisories/new](https://github.com/quaresma870/cicd-templates/security/advisories/new).

This is a solo-maintained project — expect an initial response within a
few days, not guaranteed SLAs. A confirmed vulnerability gets a fix and a
CHANGELOG entry; a genuine security-relevant fix (not a general bug) gets
a GitHub Security Advisory too.
