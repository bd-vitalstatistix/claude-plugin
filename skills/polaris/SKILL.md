---
description: Add Polaris (Black Duck) SAST+SCA security scanning to any repo
disable-model-invocation: false
---

# Polaris Security Scanning

Add Black Duck Polaris SAST and SCA scanning to a repository via GitHub Actions.

## Reference implementations
- `bd-vitalstatistix/service-mcp` — baseline (requirements.txt / pyproject.toml)
- `bd-vitalstatistix/service-llm` — UV inline script (`# /// script`) workaround
- `bd-vitalstatistix/dbt-uc-transforms` — dbt project; excludes build artefacts from capture

---

## One-time repo setup

```bash
gh variable set POLARIS_SERVER_URL \
  --body "https://polaris.blackduck.com" \
  -R <org>/<repo>

# Ask the user: "What env var holds your Polaris API token?" — do not guess.
gh secret set POLARIS_ACCESS_TOKEN \
  --body "$YOUR_TOKEN_VAR" \
  -R <org>/<repo>
```

The Polaris project is auto-created on first scan. **Run the initial scan on the default branch** — whichever branch scans first becomes the project's default branch permanently.

Portfolio: `Data Science` / Project name: the repo name.

---

## Branch and PR workflow

Never commit Polaris files to whatever branch is currently checked out. Detect the default branch first, branch off it, and open a PR.

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
git fetch origin
git checkout -b chore/add-polaris-scanning origin/$DEFAULT_BRANCH

# ... write files, commit ...

git push -u origin chore/add-polaris-scanning
gh pr create \
  --title "chore: add Polaris SAST/SCA security scanning" \
  --base "$DEFAULT_BRANCH" \
  --head chore/add-polaris-scanning \
  --body "Adds GitHub Actions workflow for SAST+SCA scanning."
```

The workflow triggers on `pull_request`, so the scan runs on the PR before files land on the default branch.

---

## GitHub Actions workflow

Create at `.github/workflows/polaris-scan.yml`. Replace `main` in the `on:` triggers with `$DEFAULT_BRANCH`.

```yaml
name: Polaris Security Scan

on:
  push:
    branches: [ main ]          # ← replace with actual default branch
  pull_request:
    branches: [ main ]          # ← replace with actual default branch
  workflow_dispatch: {}

permissions:
  contents: read
  pull-requests: write
  security-events: write
  issues: write
  statuses: write

jobs:
  polaris-scan:
    runs-on: ubuntu-latest
    name: Polaris SAST and SCA Scan

    env:
      POLARIS_SERVER_URL: ${{ vars.POLARIS_SERVER_URL }}
      POLARIS_ACCESS_TOKEN: ${{ secrets.POLARIS_ACCESS_TOKEN }}
      POLARIS_PROJECT_NAME: "<repo-name>"
      POLARIS_APPLICATION_NAME: "Data Science"
      POLARIS_BRANCH_NAME: ${{ github.head_ref || github.ref_name }}
      POLARIS_PARENT_BRANCH: ${{ github.event.base_ref || 'main' }}  # replace 'main'
      BRIDGE_GITHUB_USER_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      BRIDGE_GITHUB_REPOSITORY_OWNER_NAME: ${{ github.repository_owner }}
      BRIDGE_GITHUB_REPOSITORY_NAME: ${{ github.event.repository.name }}
      BRIDGE_GITHUB_REPOSITORY_BRANCH_NAME: ${{ github.head_ref || github.ref_name }}
      INCLUDE_DIAGNOSTICS: 'false'

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'    # omit for non-Python projects

      # REQUIRED for SCA — Detect inspects the live environment, not dep files.
      # Without this, SCA finds nothing and the project appears SAST-only.
      # Choose the right command (see language-specific notes):
      #   requirements.txt  →  pip install -r requirements.txt
      #   pyproject.toml    →  pip install .
      #   Pipfile           →  pip install pipenv && pipenv install
      #   flat-layout       →  tomllib workaround (see language-specific notes)
      - name: Install Python dependencies
        run: pip install -r requirements.txt

      # ── UV inline script workaround (remove if not needed) ─────────────────
      # Only needed if scripts/*.py use `# /// script` (PEP 723) blocks.
      # Detect cannot parse that format; export to requirements files first.
      # - name: Install uv
      #   uses: astral-sh/setup-uv@v5
      # - name: Export UV inline script dependencies
      #   run: |
      #     for script in scripts/*.py; do
      #       name=$(basename "$script" .py)
      #       uv export --script "$script" --no-hashes \
      #         --output-file "scripts/requirements-${name}.txt" 2>/dev/null || true
      #     done
      # ───────────────────────────────────────────────────────────────────────

      - name: Run Polaris Scan
        # Use continue-on-error: false during initial setup so failures are visible.
        # Revert to true once both SAST+SCA appear in the Polaris dashboard.
        continue-on-error: false
        uses: blackduck-inc/black-duck-security-scan@v2
        with:
          polaris_server_url: ${{ env.POLARIS_SERVER_URL }}
          polaris_access_token: ${{ env.POLARIS_ACCESS_TOKEN }}
          polaris_application_name: ${{ env.POLARIS_APPLICATION_NAME }}
          polaris_project_name: ${{ env.POLARIS_PROJECT_NAME }}
          polaris_branch_name: ${{ env.POLARIS_BRANCH_NAME }}
          polaris_branch_parent_name: ${{ env.POLARIS_PARENT_BRANCH }}
          polaris_assessment_types: "SAST,SCA"
          polaris_prComment_enabled: true
          github_token: ${{ secrets.GITHUB_TOKEN }}
          polaris_upload_sarif_report: true
          mark_build_status: success

      - name: Check for SARIF output
        id: check_sarif
        run: |
          if [ -f "polaris-results.sarif" ]; then
            echo "sarif_exists=true" >> $GITHUB_OUTPUT
          else
            echo "sarif_exists=false" >> $GITHUB_OUTPUT
          fi

      - name: Upload SARIF to GitHub Security tab
        if: steps.check_sarif.outputs.sarif_exists == 'true'
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: polaris-results.sarif
          category: polaris-security-scan

      - name: Archive scan results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: polaris-scan-results
          path: |
            polaris-results.sarif
            polaris-reports/
            .polaris/
          retention-days: 30
```

---

## .gitignore additions

```gitignore
# Polaris / Black Duck
polaris-results.sarif
polaris-reports/
.polaris/
polaris-output/
bridge-cli*

# UV inline script dependency exports
scripts/requirements-*.txt
```

---

## Checklist

**Setup:**
- [ ] Ask: *"What env var holds your Polaris API token?"* — do not assume
- [ ] `gh variable set` and `gh secret set` for the repo
- [ ] Detect default branch; create `chore/add-polaris-scanning` off it
- [ ] Create `.github/workflows/polaris-scan.yml` — update `POLARIS_PROJECT_NAME` and `on:` branch names
- [ ] Add the correct `pip install` step for this repo's dependency format (see language notes)
- [ ] Add `.gitignore` entries
- [ ] Commit, push, open PR

**Verification:**
- [ ] Polaris scan step is green on the PR (not skipped, not red)
- [ ] Both SAST **and** SCA appear in the Polaris dashboard for this project
- [ ] GitHub Security tab shows SARIF results
- [ ] Merge PR

**Post-merge:**
- [ ] Set `continue-on-error: true` and push to avoid blocking deployments

---

## Language-specific notes

### SCA requires installed packages

Detect inspects the **live Python environment**. If `pip install` has not run, Detect finds nothing and SCA silently produces zero results — the project shows as SAST-only with no error.

| Format | Install command |
|---|---|
| `requirements.txt` | `pip install -r requirements.txt` |
| `pyproject.toml` | `pip install .` |
| `Pipfile` | `pip install pipenv && pipenv install` |
| UV inline scripts | uv export workaround (see template comments) |

If SCA still returns nothing after installing, add `DETECT_ACCURACY_REQUIRED: NONE` to the `env:` block.

### pyproject.toml flat-layout (multiple top-level packages)

If the repo has multiple top-level directories (e.g. `app/` + `charts/`), `pip install .` fails:

```
error: Multiple top-level packages discovered in a flat-layout: ['app', 'charts'].
```

Use the tomllib workaround — parse and install deps directly, bypassing setuptools discovery:

```yaml
- name: Install Python dependencies
  run: |
    python -c "
    import tomllib, subprocess, sys
    with open('pyproject.toml', 'rb') as f:
        deps = tomllib.load(f)['project']['dependencies']
    subprocess.run([sys.executable, '-m', 'pip', 'install'] + deps, check=True)
    "
```

Alternatively, add `[tool.setuptools.packages.find]` to `pyproject.toml` if you own the file.

### Non-Python projects

Remove the Python setup and pip install steps. Adjust capture dirs in `polaris.yml` if using one.

### Makefile false-positive (C language detection)

If Polaris incorrectly detects C/C++ due to a Makefile, add `"Makefile"` to `excludePatterns` in `polaris.yml`.

---

## Troubleshooting

### SCA enters "Pending Review" state

```
ERROR: error while waiting for test with assessment type "SCA(scaPackage)":
seems the test has encountered error and has entered the state "Pending Review"
```

Bridge CLI exits with code 2. SAST may have completed fine. This state is undocumented.

1. **Retry** — most often transient, especially on first scan of a new branch: `gh workflow run polaris-scan.yml --repo <org>/<repo> --ref <branch>`
2. **Check Bridge CLI version** — v3.8.0 had a known bug; upgrade to v3.8.1+
3. **Check Polaris UI** scan history for a more detailed error on the failed SCA run
4. **File a support case** if it persists — include the scan ID from the error

### Dashboard lags after CI passes

After a successful run, the Polaris dashboard may lag several minutes. Confirm both scans actually submitted by looking for these lines in the CI log:

```
polaris.test.sast.tests.sastFull.artifacts.uploadSuccessful
polaris.test.sca.tests.scaPackage.artifacts.uploadSuccessful
```

Both present = both submitted. Dashboard will catch up.

---

## Polaris MCP server

```json
{
  "mcpServers": {
    "polaris": {
      "type": "http",
      "url": "https://polaris.blackduck.com/api/mcp",
      "headers": { "Api-Token": "${POLARIS_ACCESS_TOKEN}" }
    }
  }
}
```

Token must be in the shell environment before launching Claude Code. Header name is `Api-Token` (case-sensitive). EU/KSA: replace hostname with `eu.polaris.blackduck.com` or `ksa.polaris.blackduck.com`.
