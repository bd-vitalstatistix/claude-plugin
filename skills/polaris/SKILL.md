---
description: Add Polaris (Black Duck) SAST+SCA security scanning to any repo
disable-model-invocation: false
---

# Polaris Security Scanning

Add Black Duck Polaris SAST and SCA security scanning to a repository. Covers GitHub Actions CI integration, local development scanning, and one-time repo setup.

## Reference implementations
- `bd-vitalstatistix/service-mcp` — baseline (requirements.txt / pyproject.toml project)
- `bd-vitalstatistix/service-llm` — with UV inline script (`# /// script`) workaround
- `bd-vitalstatistix/hub-rest-api-python` — non-`main` default branch (`master`); Python SDK
- `bd-vitalstatistix/dbt-uc-transforms` — dbt project; excludes dbt build artefacts from capture
- `bd-vitalstatistix/minimal-hub-mcp` — UV inline scripts only (no requirements.txt); UV export workaround enabled

---

## One-time repo setup

Must be done once per repository before the first scan runs:

```bash
# Public variable — URL is not sensitive
gh variable set POLARIS_SERVER_URL \
  --body "https://polaris.blackduck.com" \
  -R <org>/<repo>

# Secret — ask the user: "What environment variable holds your Polaris API token?"
# Common names: POLARIS_ACCESS_TOKEN, POLARIS_API_KEY, POLARIS_TOKEN, etc.
# Use whatever variable name the user provides:
gh secret set POLARIS_ACCESS_TOKEN \
  --body "$YOUR_TOKEN_VAR" \     # replace $YOUR_TOKEN_VAR with the user's actual env var
  -R <org>/<repo>

# Verify
gh variable list -R <org>/<repo> | grep POLARIS
gh secret list -R <org>/<repo> | grep POLARIS
```

**Before running the secret command**, ask the user: *"What environment variable in your shell holds your Polaris API token?"* Different team members use different variable names. Do not guess or hardcode one.

The Polaris project is auto-created on first scan if your token has sufficient permissions. **Run the initial scan on the default branch** — whichever branch scans first becomes the Polaris project's default branch and cannot be changed via the UI.

The project should go under:
- **Portfolio / Application**: `Data Science`
- **Project name**: the repo name (e.g. `service-llm`, `service-mcp`)

---

## Branch and PR workflow — IMPORTANT

**Never commit Polaris files directly to whatever branch is currently checked out.** Repos use different default branch names (`main`, `master`, or something else entirely), and the agent may be invoked while a feature branch is active.

### Step 1 — detect the default branch

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
echo "Default branch: $DEFAULT_BRANCH"
```

### Step 2 — create a fresh branch off the default

```bash
git fetch origin
git checkout -b chore/add-polaris-scanning origin/$DEFAULT_BRANCH
```

### Step 3 — write all Polaris files, commit, push, open PR

```bash
# ... write polaris.yml, .github/workflows/polaris-scan.yml, .gitignore updates ...

git add polaris.yml .github/workflows/polaris-scan.yml .gitignore
git commit -m "chore: add Polaris SAST/SCA scanning"
git push -u origin chore/add-polaris-scanning

gh pr create \
  --title "chore: add Polaris SAST/SCA security scanning" \
  --base "$DEFAULT_BRANCH" \
  --head chore/add-polaris-scanning \
  --body "Adds polaris.yml and GitHub Actions workflow for SAST+SCA scanning."
```

**Why this matters:**
- The workflow's `on.push.branches` and `on.pull_request.branches` must match the actual default branch name
- Polaris's first scan sets the project's default branch permanently — it must be the real default, not a feature branch
- PRs let the scan run before the files land on the default branch (the workflow triggers on pull_request too)

---

## polaris.yml (repo root)

Create at the repo root. Adjust `project.name` and `capture.fileSystem.captureDirs` for the target repo.

```yaml
project:
  name: "<repo-name>"

application:
  name: "Data Science"

branch:
  name: "${POLARIS_BRANCH_NAME:main}"
  parent:
    name: "${POLARIS_PARENT_BRANCH:main}"

server:
  url: "${POLARIS_SERVER_URL:https://polaris.blackduck.com}"

assessment:
  types: ["SAST", "SCA"]

access:
  token: "${POLARIS_ACCESS_TOKEN}"

analysis:
  mode: "cloud"

capture:
  fileSystem:
    captureDirs:
      - "src/"        # adjust to the actual source directories
      - "app/"
      - "scripts/"
    excludePatterns:
      - "**/.git/**"
      - "**/__pycache__/**"
      - "**/venv/**"
      - "**/.venv/**"
      - "**/node_modules/**"
      - "**/bridge-cli*/**"
      - "**/polaris-output/**"
      - "**/polaris-reports/**"

  python:
    includeRequirementsFiles: true
    requirementsFiles:
      - "requirements.txt"
      - "pyproject.toml"
      # If the repo uses UV inline scripts, add:
      # - "scripts/requirements-*.txt"

reporting:
  sarif:
    create: true
    file: "polaris-results.sarif"
  pdf:
    create: true
    path: "./polaris-reports/"

prComment:
  enabled: true

upload:
  localConfig: true

timeout:
  analysis: 3600
```

---

## GitHub Actions workflow

Create at `.github/workflows/polaris-scan.yml`.

**Important**: Replace `main` in the `on:` trigger block with the repo's actual default branch name (detected in step 1 above). Do not hardcode `main` — the repo may use `master` or another name.

```yaml
name: Polaris Security Scan

on:
  push:
    branches: [ main ]   # ← replace with actual default branch ($DEFAULT_BRANCH)
  pull_request:
    branches: [ main ]   # ← replace with actual default branch ($DEFAULT_BRANCH)
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
      POLARIS_PARENT_BRANCH: ${{ github.event.base_ref || 'main' }}  # replace 'main' with actual default branch
      BRIDGE_GITHUB_USER_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      BRIDGE_GITHUB_REPOSITORY_OWNER_NAME: ${{ github.repository_owner }}
      BRIDGE_GITHUB_REPOSITORY_NAME: ${{ github.event.repository.name }}
      BRIDGE_GITHUB_REPOSITORY_BRANCH_NAME: ${{ github.head_ref || github.ref_name }}
      INCLUDE_DIAGNOSTICS: 'false'

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Setup Python          # omit if not a Python project
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      # REQUIRED for SCA: Detect inspects the live Python environment.
      # Without this step, SCA finds nothing and the project shows SAST-only.
      # Adjust command for your dependency format:
      #   requirements.txt  →  pip install -r requirements.txt
      #   pyproject.toml    →  pip install .
      #   Pipfile           →  pip install pipenv && pipenv install
      # Omit entirely for non-Python projects.
      - name: Install Python dependencies
        run: pip install -r requirements.txt

      # ── UV inline script workaround ────────────────────────────────────────
      # Only needed if scripts/*.py use `# /// script` (PEP 723) dependency
      # blocks. Black Duck Detect cannot parse that format; exporting to
      # sibling requirements files makes them visible to the pip inspector.
      # Remove this step for repos that use requirements.txt / pyproject.toml.
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
        # continue-on-error: false during setup — makes auth/config failures visible (red step)
        # Once scan is verified working (both SAST+SCA appear in Polaris dashboard),
        # revert to: continue-on-error: true
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

## Local scanning script

Create at `scripts/polaris-scan.sh` (chmod +x). Downloads Bridge CLI automatically if not present.

```bash
#!/bin/bash
# Local Polaris scan — usage: ./scripts/polaris-scan.sh [sast|sca|both]
set -e

SCAN_TYPE="${1:-both}"
PROJECT_NAME="${POLARIS_PROJECT_NAME:-$(basename "$(git rev-parse --show-toplevel)")}"
APPLICATION_NAME="${POLARIS_APPLICATION_NAME:-Data Science}"
BRANCH_NAME="${POLARIS_BRANCH_NAME:-$(git rev-parse --abbrev-ref HEAD)}"

# Validate env
for var in POLARIS_SERVER_URL POLARIS_ACCESS_TOKEN; do
  [[ -z "${!var}" ]] && { echo "❌ $var not set"; exit 1; }
done

# Load .env if present
[[ -f ".env" ]] && { set -a; source .env; set +a; }

# Resolve assessment types
case "$SCAN_TYPE" in
  sast) ASSESSMENT_TYPES="SAST" ;;
  sca)  ASSESSMENT_TYPES="SCA"  ;;
  both) ASSESSMENT_TYPES="SAST,SCA" ;;
  *)    echo "Usage: $0 [sast|sca|both]"; exit 1 ;;
esac

# Find or download Bridge CLI
BRIDGE_CLI=""
if command -v bridge-cli &>/dev/null; then
  BRIDGE_CLI="bridge-cli"
elif [[ -f "./bridge-cli/bridge-cli" ]]; then
  BRIDGE_CLI="./bridge-cli/bridge-cli"
else
  OS=$(uname -s | tr '[:upper:]' '[:lower:]')
  [[ "$OS" == "darwin" ]] && \
    URL="https://repo.blackduck.com/bds-integrations-release/com/blackduck/integration/bridge/binaries/bridge-cli-bundle/latest/bridge-cli-bundle-macosx.zip" || \
    URL="https://repo.blackduck.com/bds-integrations-release/com/blackduck/integration/bridge/binaries/bridge-cli-bundle/latest/bridge-cli-bundle-linux64.zip"
  curl -fLsS -o bridge-cli.zip "$URL"
  unzip -q bridge-cli.zip -d bridge-cli-temp
  BRIDGE_CLI=$(find bridge-cli-temp -name "bridge-cli" -type f | head -1)
  chmod +x "$BRIDGE_CLI"
fi

# Better Python detection for SCA
[[ -f "uv.lock" ]] && export PATH="$(pwd)/.venv/bin:$PATH"
[[ -d ".venv" ]] && source .venv/bin/activate 2>/dev/null || true
export DETECT_ACCURACY_REQUIRED=NONE

mkdir -p ./polaris-output

"$BRIDGE_CLI" --stage polaris \
  polaris.serverUrl="$POLARIS_SERVER_URL" \
  polaris.accessToken="$POLARIS_ACCESS_TOKEN" \
  polaris.project.name="$PROJECT_NAME" \
  polaris.application.name="$APPLICATION_NAME" \
  polaris.branch.name="$BRANCH_NAME" \
  polaris.assessment.types="$ASSESSMENT_TYPES" \
  polaris.reports.sarif.create=true \
  polaris.reports.sarif.file.path="./polaris-output/polaris-results.sarif"

[[ -d "bridge-cli-temp" ]] && rm -rf bridge-cli-temp bridge-cli.zip
echo "✅ Done — results at ./polaris-output/ and https://polaris.blackduck.com"
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

# UV inline script dependency exports (if applicable)
scripts/requirements-*.txt
```

---

## Checklist for a new repo

**Setup:**
- [ ] Ask the user: *"What environment variable holds your Polaris API token?"* — do not assume
- [ ] Run repo setup commands (variable + secret) via `gh` using the user's token variable
- [ ] Detect the default branch: `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'`
- [ ] Create a fresh branch: `git fetch origin && git checkout -b chore/add-polaris-scanning origin/$DEFAULT_BRANCH`
- [ ] Create `polaris.yml` at repo root — update `project.name` and `captureDirs`
- [ ] Create `.github/workflows/polaris-scan.yml` — update `POLARIS_PROJECT_NAME` and replace `main` in `on:` triggers with actual default branch
- [ ] For Python repos: confirm the correct `pip install` step is present before the scan step (see language-specific notes)
- [ ] Leave `continue-on-error: false` for initial testing (default in template)
- [ ] Add `scripts/polaris-scan.sh` if local scanning is wanted
- [ ] Add `.gitignore` entries
- [ ] Commit, push branch, open PR targeting `$DEFAULT_BRANCH` — the scan runs on the PR itself

**Verification:**
- [ ] Verify the Polaris scan check passes on the PR (green, not skipped)
- [ ] Confirm **both SAST and SCA** appear in the Polaris dashboard for this project (not just SAST)
- [ ] Check GitHub Security tab for SARIF results after scan completes
- [ ] Merge PR — this triggers the first scan on the default branch

**Post-verification:**
- [ ] Once scan is confirmed working: set `continue-on-error: true` in the workflow and push to avoid blocking deployments

---

## Language-specific notes

### SCA requires installed packages — CRITICAL

Black Duck Detect (the SCA component) inspects the **live Python environment**, not just dependency files. If `pip install` has not run before the scan step, Detect finds no installed packages and SCA produces zero findings — even if `polaris_assessment_types: "SAST,SCA"` is correctly set. The project will appear as SAST-only in the Polaris dashboard.

**Always install dependencies before the scan step.** Use the appropriate command for the repo's dependency format:

| Dependency format | Install command |
|---|---|
| `requirements.txt` | `pip install -r requirements.txt` |
| `pyproject.toml` (no requirements.txt) | `pip install .` |
| `Pipfile` | `pip install pipenv && pipenv install` |
| UV inline scripts (`# /// script`) | uv export workaround (see below) |
| JS/TS — no Python | omit Python setup steps entirely |

If SCA still returns no results after installing, add `DETECT_ACCURACY_REQUIRED: NONE` to the workflow env block — this allows Detect to proceed even when pip cannot fully resolve all packages.

### `continue-on-error` lifecycle

The scan step has a `continue-on-error` flag that controls whether a scan failure blocks the workflow:

- **During initial setup / testing**: set `continue-on-error: false` — failures (auth errors, misconfig, SCA finding nothing) will make the workflow step visibly red. This is intentional: you need to see failures while verifying the setup.
- **Once verified working**: revert to `continue-on-error: true` — scan failures won't block deployments. This is the appropriate production default for teams without strict security gates.

The templates in this skill default to `continue-on-error: false`. **Revert to `true` once you've confirmed a clean scan run** with both SAST and SCA results appearing in the Polaris dashboard.

### Python with requirements.txt / pyproject.toml
Standard setup — add `pip install -r requirements.txt` or `pip install .` before the scan step. Keep the `capture.python.requirementsFiles` list in `polaris.yml` pointing at the right files.

### Python with UV inline scripts (`# /// script`)
Black Duck Detect cannot parse PEP 723 inline dependency blocks. Add the uv export step to the workflow (shown commented-out above) and add `scripts/requirements-*.txt` to both `polaris.yml` → `requirementsFiles` and `.gitignore`.

### Non-Python projects
Remove the Python setup and pip install steps from the workflow. Adjust `captureDirs` in `polaris.yml` to point at source directories. The `capture.python` block can be omitted entirely.

### Python with pyproject.toml flat-layout (multiple top-level dirs)

If the repo has multiple top-level directories that look like Python packages (e.g. `app/` for code and `charts/` for Helm), `pip install .` will fail with:

```
error: Multiple top-level packages discovered in a flat-layout: ['app', 'charts'].
```

This is a setuptools constraint — it refuses to guess which directory is the real package. **This is not documented in Black Duck/Polaris docs** and surfaces only when SCA triggers a pip install.

Two fixes, in order of preference:

**Option A — tomllib workaround** (no changes to `pyproject.toml`): parse and install deps directly, bypassing setuptools discovery:

```yaml
- name: Install Python dependencies
  # pip install . fails due to multi-package flat layout.
  # Parse dependencies directly from pyproject.toml so SCA can inspect them.
  run: |
    python -c "
    import tomllib, subprocess, sys
    with open('pyproject.toml', 'rb') as f:
        deps = tomllib.load(f)['project']['dependencies']
    subprocess.run([sys.executable, '-m', 'pip', 'install'] + deps, check=True)
    "
```

**Option B — fix `pyproject.toml`**: add an explicit package declaration so setuptools knows what to install:

```toml
[tool.setuptools.packages.find]
where = ["."]
include = ["app*"]   # or whatever the actual Python package directory is
```

Option A is preferred when you don't own the `pyproject.toml` or want to avoid touching project metadata.

### Makefile detection false-positive (C language)
If Polaris incorrectly detects C/C++ due to a Makefile, add `"Makefile"` to `excludePatterns` in `polaris.yml` (see service-mcp for this pattern).

---

## Bridge CLI download URLs (current as of 2026)

```
Linux x86_64: https://repo.blackduck.com/bds-integrations-release/com/blackduck/integration/bridge/binaries/bridge-cli-bundle/latest/bridge-cli-bundle-linux64.zip
macOS (arm64/x86): https://repo.blackduck.com/bds-integrations-release/com/blackduck/integration/bridge/binaries/bridge-cli-bundle/latest/bridge-cli-bundle-macosx.zip
```

Note: The old `sig-repo.synopsys.com` domain is deprecated — use `repo.blackduck.com`.

---

## Viewing results

- **Polaris web UI**: https://polaris.blackduck.com → select portfolio "Data Science" → project
- **GitHub Security tab**: repo → Security → Code scanning alerts (populated from SARIF upload)
- **PR comments**: Polaris posts a findings summary comment on each PR automatically
- **Polaris MCP**: see below

---

## Polaris MCP server

The Polaris Issue Management MCP server is hosted at `https://polaris.blackduck.com/api/mcp`. It exposes read-only tools for querying issues, portfolio metrics, and entitlements — no installation required.

### `.mcp.json` (repo root)

```json
{
  "mcpServers": {
    "polaris": {
      "type": "http",
      "url": "https://polaris.blackduck.com/api/mcp",
      "headers": {
        "Api-Token": "${POLARIS_ACCESS_TOKEN}"
      }
    }
  }
}
```

- Auth uses the same `POLARIS_ACCESS_TOKEN` as the scan. The header name is `Api-Token` (case-sensitive).
- Token must be set in the shell environment before launching Claude Code — it is not read from `.env` automatically.
- For EU/KSA instances, replace the URL with `https://eu.polaris.blackduck.com/api/mcp` or `https://ksa.polaris.blackduck.com/api/mcp`.

The server is self-documenting — Claude Code will list available tools on connection.

### Checklist addition

Add `.mcp.json` to the new-repo checklist and verify the MCP server connects:

```bash
# Quick connectivity check — should return tool list
curl -s -H "Api-Token: $POLARIS_ACCESS_TOKEN" \
  https://polaris.blackduck.com/api/mcp | head -20
```

---

## Troubleshooting

### SCA produces zero findings / project shows SAST-only

The most common cause: dependencies were not installed before the scan step. Black Duck Detect inspects the **live Python environment** — if no packages are installed, it finds nothing and silently skips SCA.

Checklist:
1. Confirm a `pip install` step exists **before** the Polaris scan step
2. Confirm the install step uses the correct format for the repo (see language-specific notes)
3. Add `DETECT_ACCURACY_REQUIRED: NONE` to the workflow env block if Detect is failing to resolve packages
4. For `pyproject.toml` repos with multiple top-level dirs — see flat-layout workaround above

### SCA scan enters "Pending Review" state

Symptom (Bridge CLI output):
```
ERROR: error while waiting for test with assessment type "SCA(scaPackage)" and id "...":
seems the test has encountered error and has entered the state "Pending Review"
```
Bridge CLI exits with code 2 ("Adapter failed"). The workflow step fails even though SAST completed fine.

**This state is not documented in official Black Duck/Polaris docs.** A related undocumented state ("Pending SNPS") has identical symptoms.

Resolution steps:
1. **Retry the run** — this is most often a transient server-side failure, especially on first-scan of a new branch. Re-trigger via `gh workflow run polaris-scan.yml --repo <org>/<repo> --ref <branch>`
2. **Check Bridge CLI version** — v3.8.0 had a known bug causing similar failures; upgrade to v3.8.1+
3. **Check the Polaris UI** — navigate to the project's scan history and inspect the failed SCA scan for a more detailed error message
4. **File a support case** if the issue persists across retries — include the scan ID from the error message

### Polaris dashboard shows branch data lagging after CI passes

After a successful CI run (both SAST and SCA artifacts uploaded), the Polaris dashboard may still show only main branch data for several minutes. This is normal — server-side analysis runs asynchronously after artifact upload.

To confirm a scan actually ran, check the CI log for these lines rather than waiting for the dashboard:
```
polaris.test.sast.tests.sastFull.artifacts.uploadSuccessful
polaris.test.sca.tests.scaPackage.artifacts.uploadSuccessful
```
Both lines present = both scans submitted successfully. Dashboard will catch up.
