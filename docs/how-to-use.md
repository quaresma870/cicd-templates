# How to Use These Templates

## Quick start

### 1. Copy the template

```bash
# In your project repo
mkdir -p .github/workflows
cp /path/to/cicd-templates/templates/python/ci.yml .github/workflows/ci.yml
```

Or download directly from GitHub:

```bash
curl -o .github/workflows/ci.yml \
  https://raw.githubusercontent.com/quaresma870/cicd-templates/main/templates/python/ci.yml
```

### 2. Configure secrets and variables

Go to your repo → **Settings → Secrets and variables → Actions**

See [secrets-setup.md](secrets-setup.md) for the full list per template.

### 3. Customise the template

Each template has clearly marked sections you'll typically want to change:

| Section | What to customise |
|---------|-------------------|
| `env.PYTHON_VERSION` | Python/Node version |
| `env.IMAGE_NAME` | Docker image name fallback |
| `test` job | Your actual test command |
| `deploy-vps` → `script` | Deploy commands on your server |
| `on.push.branches` | Which branches trigger the pipeline |

### 4. Push and watch

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add CI/CD pipeline"
git push
```

The pipeline starts automatically. Check **Actions** tab in your repo.

---

## Adapting the generic template

The `generic/ci.yml` has placeholder steps for the test stage. Replace the placeholder with your actual command:

```yaml
# Before
- name: Run tests (customise this step)
  run: echo "Add your test command here"

# After (Python example)
- uses: actions/setup-python@v6
  with:
    python-version: "3.11"
    cache: pip
- run: pip install -r requirements.txt && pytest tests/ -v
```

---

## Disabling stages you don't need

Comment out or delete entire jobs. Example — removing the VPS deploy:

```yaml
# Delete or comment out the deploy-vps job entirely
# deploy-vps:
#   name: Deploy to VPS
#   ...
```

And remove it from `needs:` in any downstream job.

---

## Running manually

All templates support `workflow_dispatch` for manual runs:

1. Go to **Actions** tab in your repo
2. Select the workflow
3. Click **Run workflow**
4. Fill in the inputs (deploy target, dry-run mode, etc.)

---

## Combining templates

You can run multiple workflows in the same repo. Example — add the security audit alongside your main pipeline:

```
.github/workflows/
├── ci.yml          ← copied from templates/python/ci.yml
└── security.yml    ← copied from templates/security/ci.yml
```

The security pipeline runs on a weekly schedule independently of the main CI.

---

## Versioning your pipeline

Pin actions to specific versions for stability. All templates already use pinned versions. To update:

```bash
# Check for newer versions
grep "uses:" .github/workflows/ci.yml

# Update manually or use Dependabot
```

### Enabling Dependabot for Actions

Add to `.github/dependabot.yml` in your repo:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```
