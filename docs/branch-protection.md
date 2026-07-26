# Branch Protection

Every template in this repo assumes **deploy only happens on `main`** — PRs
run lint/test/build but never deploy (see the README's "Common features").
That assumption is only actually safe if `main` is protected: without it,
anyone with push access can bypass the pipeline entirely by pushing straight
to `main`, or merge a PR whose checks never passed.

This is a guide for repos that **adopt** one of these templates (copied or
via a reusable workflow) — not for this `cicd-templates` repo itself, which
has its own protection configured separately.

## Setting it up

Go to: **Settings → Branches → Add branch protection rule** (or **Rulesets**
on newer GitHub UIs) in the repo that adopted the template, and target `main`.

### Required status checks

Enable **"Require status checks to pass before merging"**, then add the job
names from whichever template you adopted. These are the `name:` fields
under `jobs:` in the workflow — for example, for `templates/python/ci.yml`:

| Required check | From job |
|---|---|
| `Lint` | `jobs.lint` |
| `Test` | `jobs.test` |
| `Security Scan` | `jobs.security` |
| `Build Docker image` | `jobs.build` |

Don't require `Deploy to VPS` — it only runs on `main` pushes, never on a
PR, so requiring it would make every PR permanently unmergeable.

If you're calling a reusable workflow (Option B in the README) instead of
copying the template, the check names are prefixed with the calling job's
own name (e.g. `ci / Lint`) — check the **Checks** tab on an actual PR to
get the exact string GitHub uses before typing it into the required-checks
list.

Also enable **"Require branches to be up to date before merging"** — without
it, a PR can merge with checks that passed against a stale base, not what's
actually about to land on `main`.

### Other settings worth turning on

- **Require a pull request before merging** — otherwise the required checks
  above can still be bypassed by a direct push.
- **Require review from Code Owners** — pairs with `.github/CODEOWNERS` if
  the adopting repo has one (this repo does; see `CONTRIBUTING.md`).
- **Do not allow bypassing the above settings** — without this, repo admins
  (often everyone on a small team) can still push straight past every rule
  above. Decide deliberately whether that's wanted, don't leave it as the
  default.

### If you're using the Dependabot auto-merge pattern

`.github/workflows/dependabot-auto-merge.yml` (see this repo's own copy for
the pattern) relies on **"Allow auto-merge"** under Settings → General,
*separate* from branch protection. `gh pr merge --auto` only actually merges
once your required status checks above report success — auto-merge doesn't
bypass branch protection, it waits on it.

## Protecting tags (optional)

If you adopt this repo's `release.yml` pattern (a `vX.Y.Z` tag push creates
a GitHub Release), consider a tag protection rule — **Settings → Tags → New
rule**, pattern `v*` — so only people with the right permissions can push a
release tag.
