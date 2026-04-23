---
description: Add Polaris (Black Duck) SAST+SCA security scanning to any repo
disable-model-invocation: false
---

# Polaris Security Scanning

Add Black Duck Polaris SAST and SCA security scanning to a repository. Covers GitHub Actions CI integration, local development scanning, and one-time repo setup.

## Reference implementations
- `bd-vitalstatistix/service-mcp` — baseline (requirements.txt / pyproject.toml project)
- `bd-vitalstatistix/service-llm` — with UV inline script (`# /// script`) workaround

---

## One-time repo setup

Must be done once per repository before the first scan runs:

```bash
# Public variable — URL is not sensitive
gh variable set POLARIS_SERVER_URL \
  --body "https://polaris.blackduck.com" \
  -R <org>/<repo>

# Secret — sourced from your local shell
gh secret set POLARIS_ACCESS_TOKEN \
  --body "$POLARIS_API_KEY_PROD" \
  -R <org>/<repo>

# Verify
gh variable list -R <org>/<repo> | grep POLARIS
gh secret list -R <org>/<repo> | grep POLARIS
```

The Polaris project is auto-created on first scan if your token has sufficient permissions. **Run the initial scan on `main`** — whichever branch scans first becomes the Polaris project's default branch and cannot be changed via the UI.

The project should go under:
- **Portfolio / Application**: `Data Science`
- **Project name**: the repo name (e.g. `service-llm`, `service-mcp`)

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

```yaml
name: Polaris Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
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
      POLARIS_PARENT_BRANCH: ${{ github.event.base_ref || 'main' }}
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

      - name: Install Python dependencies   # omit if no requirements.txt
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
        continue-on-error: true
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

- [ ] Run repo setup commands (variable + secret) via `gh`
- [ ] Create `polaris.yml` at repo root — update `project.name` and `captureDirs`
- [ ] Create `.github/workflows/polaris-scan.yml` — update `POLARIS_PROJECT_NAME`
- [ ] Add `scripts/polaris-scan.sh` if local scanning is wanted
- [ ] Add `.gitignore` entries
- [ ] Push to `main` to trigger the first scan — verify project appears in Polaris UI
- [ ] Check GitHub Security tab for SARIF results after scan completes

---

## Language-specific notes

### Python with requirements.txt / pyproject.toml
Standard setup — no special handling needed. Keep the `capture.python.requirementsFiles` list in `polaris.yml` pointing at the right files.

### Python with UV inline scripts (`# /// script`)
Black Duck Detect cannot parse PEP 723 inline dependency blocks. Add the uv export step to the workflow (shown commented-out above) and add `scripts/requirements-*.txt` to both `polaris.yml` → `requirementsFiles` and `.gitignore`.

### Non-Python projects
Remove the Python setup and pip install steps from the workflow. Adjust `captureDirs` in `polaris.yml` to point at source directories. The `capture.python` block can be omitted entirely.

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
- **Polaris MCP**: if `POLARIS_ACCESS_TOKEN` is available, the Polaris Issue Management MCP server at `https://polaris.blackduck.com/api/mcp` exposes tools for querying issues, portfolio metrics, and entitlements
