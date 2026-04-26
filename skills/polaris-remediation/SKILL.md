---
description: Iteratively triage and remediate Polaris security findings — fix real issues on validated PR branches, record false positives for later triage
disable-model-invocation: false
---

# Polaris Remediation

Work through Polaris security findings in a repository: group them into thematic families, fix real issues on per-family PR branches (with rollback on validation failure), and record false positives for later triage.

## Prerequisites

- Polaris MCP configured in `.mcp.json` (same stanza as the polaris setup skill)
- Polaris has already scanned this repo at least once
- A clean working tree (no uncommitted changes)

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

---

## Phase 0 — Resolve project identity

```bash
# Confirm you're in the right repo
git remote get-url origin

# Detect default branch — never work on it directly
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
```

Use the Polaris MCP tools to locate the project:

1. `get_portfolio_id` → portfolio UUID
2. `get_application_dashboard` with `filter='portfolioItemName=ilike=("<application-name>")'` → application UUID
3. `get_project_dashboard` with `portfolioItemId` → project UUID for this repo
4. `get_branches` with `filter='defaultFlag==true'` → confirm which branch is the Polaris default

Record: `portfolioId`, `applicationId`, `projectId`, `branchId` (default branch).

---

## Phase 0.5 — Check for bulk false-positive directories

Before triaging individual issues, detect whether large clusters of findings come from files that should never be scanned (vendored API specs, archived docs, generated files). If so, fix the scan scope first — this avoids triaging hundreds of findings that will disappear once the exclusion lands.

### Detection

Check the total issue count. If it is much higher (5x+) than expected for the codebase size, look for a concentration pattern:

```
list_issues(
  projectId = <projectId>,
  branchId = <branchId>,
  filter = "triage:status=in=('not-dismissed','review-pending')",
  includeOccurrenceProperties = true,
  first = 50,
  sort = ["occurrence:severity|desc"]
)
```

Scan the `location` / `filename` fields. If many issues share a common path prefix that looks like generated or archived content (e.g., `public/specs/`, `docs/archive/`, `vendor/`, `third_party/`) those are candidates for exclusion from the scanner — not individual triage.

Cross-check: read the actual files at those paths. API spec YAML/JSON files, archived markdown, or generated code are high-confidence exclusion candidates.

### Fix — `coverity.yml` `capture.files.exclude-regex`

The correct mechanism for excluding directory trees from Polaris SAST (Coverity + Sigma) in hybrid/local capture mode is `coverity.yml` at the repo root, under `capture.files`. **This is the only exclusion that reliably flows through to all tools including Sigma (Black Duck Rapid Scan Static).**

Key learnings:
- `project_source_excludes` in the GitHub Actions workflow only applies to SOURCE_UPLOAD zip mode, not hybrid/local capture mode — it has no effect on sigma in typical CI setups.
- `.sigma-config.yml` auto-detection is bypassed when Bridge passes `--coverity-config /tmp/...` to `cov-run-sigma` — sigma's own config file is not used.
- `coverity.yml` `capture.files.exclude-glob` is a single `string` (not an array) and `**` glob is not supported by Coverity's glob implementation.
- `exclude-regex` supports combining multiple paths with `|` and is the reliable approach.

**Working pattern** (add to `coverity.yml`):

```yaml
capture:
  files:
    exclude-regex: "(docs/archive/|public/specs/|vendor/)"
```

The regex is matched against each file path. Any path containing the pattern is excluded from capture — and therefore from both Coverity SAST analysis and Sigma rapid scan.

If `coverity.yml` does not exist, create it at the repo root.

After pushing this change, wait for the next CI scan to complete, then re-query the branch issue count to confirm the excluded-directory issues have dropped to zero.

**Do not proceed to per-issue triage until the scan-scope exclusion is confirmed working.** Triaging issues that will disappear is wasted effort.

---

## Phase 1 — Discover validation capability

Before touching any code, determine what "validated" means in this repo. Read these files if they exist:

| File | What to look for |
|---|---|
| `Makefile` | `test`, `lint`, `check`, `typecheck` targets |
| `package.json` | `scripts.test`, `scripts.lint`, `scripts.typecheck` |
| `pyproject.toml` / `setup.cfg` | `[tool.pytest]`, `[tool.mypy]` |
| `Cargo.toml` | `cargo test`, `cargo check` |
| `.github/workflows/*.yml` | what CI runs on PRs |

**Assign a validation level** (record this and show it to the user before proceeding):

| Level | Meaning | Command |
|---|---|---|
| `full` | Test suite available and runnable | `make test` / `npm test` / `cargo test` / etc. |
| `lint` | Linter/type-checker only, no test suite | `make lint` / `npm run lint` / `mypy .` / etc. |
| `syntax` | No discoverable test/lint — syntax check only | language-specific parse-only invocation |
| `none` | Cannot validate — **do not fix, only log FPs** | — |

If validation level is `none`, proceed only with false-positive identification and logging. Do not attempt code fixes.

---

## Phase 2 — Fetch and group issues

Fetch open issues from Polaris MCP:

```
list_issues(
  projectId = <projectId>,
  branchId = <branchId>,
  filter = "triage:status=in=('not-dismissed','review-pending')",
  includeOccurrenceProperties = true,
  includeTriageProperties = true,
  includeContext = true,
  sort = ["occurrence:severity|desc"],
  first = 10   # paginate — use cursor for subsequent pages
)
```

### Grouping into families

A **family** is a set of issues that are thematically related enough that a single reviewer can assess them together in one PR. Grouping signals (use judgment, not rigid rules):

| Signal | Example family |
|---|---|
| Same checker, same directory | All `FORWARD_NULL` in `src/auth/` |
| Same root cause pattern | Null-check gaps all tracing to one API response type |
| Same file cluster | Three related issues across two files in the same feature |
| Same infrastructure component | All container capability issues in `docker-compose*.yaml` |
| Issues in test files only | All findings in `*.test.*` / `*.spec.*` (likely FPs — group together for bulk triage) |

Do not group across severity levels unless the issues are genuinely the same root cause.

**Name each family** with a slug: `forward-null-auth`, `container-net-raw`, `deadcode-test-files`, etc.

---

## Phase 3 — Per-family fix loop

For each family, run this loop:

### 3a — Assess the family

For each issue in the family:

1. Use `get_issue` with `includeOccurrenceProperties=true`, `includeType=true` (get remediation guidance)
2. Read the source file at the reported filename and line
3. Classify:
   - **False Positive**: finding is not a real risk in context (test file, intentional config, already-guarded path, infrastructure-only)
   - **Fixable**: genuine issue, fix is clear from reading the code
   - **Needs Review**: cannot determine without more context

If the entire family is false positives → skip to step 3e (log FPs, no branch needed).
If any issue is fixable → proceed to 3b.

### 3b — Create a PR branch

```bash
git fetch origin
git checkout -b fix/polaris-<family-slug> origin/$DEFAULT_BRANCH
```

Record the base commit:

```bash
BASE_COMMIT=$(git rev-parse HEAD)
```

### 3c — Fix and validate (per issue, with rollback)

For each fixable issue in the family:

1. **Read** the file at the reported line
2. **Apply the fix** using the Edit tool — only change what is necessary to address the finding
3. **Run validation** using the command identified in Phase 1
4. **On validation pass**: stage and commit
   ```bash
   git add <changed-files>
   git commit -m "fix(security): <checker> in <filename> — <one-line description>"
   ```
5. **On validation fail**: roll back to the last committed state — do not commit broken code
   ```bash
   git checkout -- <changed-files>
   ```
   Log the issue as `ATTEMPTED-ROLLBACK` (see Phase 4 log format).

If validation level is `lint` or `syntax`, apply the same pass/fail logic with the appropriate command.

After all fixable issues in the family are attempted (some may have been rolled back):

- If at least one commit was made → continue to 3d
- If all fixes were rolled back → delete the branch, log all as `ATTEMPTED-ROLLBACK`

### 3d — Open a PR

```bash
git push -u origin fix/polaris-<family-slug>

gh pr create \
  --title "fix(security): <family-slug> Polaris findings" \
  --base "$DEFAULT_BRANCH" \
  --head fix/polaris-<family-slug> \
  --body "$(cat <<'EOF'
## Polaris security remediation — <family-slug>

**Checker(s)**: <list>
**Severity**: <level>
**Files changed**: <list>

### Issues addressed
<table of issue ID, checker, filename, line, fix description>

### Issues in this family not fixed
<table of issue ID, checker, filename, line, reason (ROLLED-BACK / FALSE-POSITIVE / NEEDS-REVIEW)>

### Validation
Validation level: `<full|lint|syntax>`
Command: `<command run>`

---
🤖 Generated by polaris-remediation skill
EOF
)"
```

### 3e — Log false positives

Append to `polaris-fp-log.md` in the repo root (add to `.gitignore` if not already present):

```markdown
## <YYYY-MM-DD> — <family-slug>

| Issue ID | Checker | Severity | File | Line | Status | Reason |
|---|---|---|---|---|---|---|
| <id> | <checker> | <high/med/low> | <filename> | <line> | FALSE-POSITIVE | <reason> |
| <id> | <checker> | <severity> | <filename> | <line> | NEEDS-REVIEW | <reason> |
| <id> | <checker> | <severity> | <filename> | <line> | ATTEMPTED-ROLLBACK | Validation failed after fix attempt |
```

**Status values**:
- `FALSE-POSITIVE` — finding is not a real risk in this context
- `NEEDS-REVIEW` — insufficient context to decide; human review required
- `ATTEMPTED-ROLLBACK` — fix was attempted but validation failed; issue remains open in Polaris

Add to `.gitignore`:

```gitignore
# Polaris local triage log
polaris-fp-log.md
```

---

## Phase 4 — Progress report and next iteration

After each family is complete, report:

```
Family: forward-null-auth
  Fixed and committed: 3 issues
  Rolled back: 1 issue (validation failed)
  False positives logged: 2 issues
  PR: https://github.com/org/repo/pull/123

Remaining families: 4
Next: container-net-raw (5 issues, all high severity)
```

Pause here — confirm with the user before continuing to the next family, or proceed automatically if the user asked for unattended mode.

Use cursor pagination from `list_issues` to ensure all issues across all pages are processed before declaring done.

---

## Checklist

**Before starting:**
- [ ] Working tree is clean (`git status`)
- [ ] Polaris MCP configured and connected
- [ ] Portfolio / application / project IDs resolved
- [ ] Validation level determined and shown to user

**Per family:**
- [ ] Issues fetched and classified (FP / fixable / needs-review)
- [ ] Branch created off default branch (if any fixes to attempt)
- [ ] Each fix validated before committing; rolled back on failure
- [ ] PR opened (if any commits succeeded)
- [ ] False positives and rollbacks logged to `polaris-fp-log.md`
- [ ] Progress report shown before next family

**Constraints (never violate):**
- [ ] Never commit to the default branch
- [ ] Never commit a failing fix — rollback to last good state
- [ ] Never open a PR with zero committed fixes
- [ ] Never prescribe a fix without reading the source file first

---

## Notes on the Polaris MCP

The Polaris MCP is **read-only** — it can query issues, triage status, and remediation guidance, but cannot update triage status or dismiss issues. Triage decisions are tracked locally in `polaris-fp-log.md`. After review, a human can dismiss the logged issues in the Polaris web UI or via the Polaris API.

If and when write tools become available in the Polaris MCP, the false-positive logging step should be extended to call `updateIssueTriage` with `dismissal-reason: false-positive` and `status: dismissed` for confirmed FPs.

---

## Reference implementations

### bd-assist (SIG-Innovation-Zone/bd-assist)

**Initial state**: 1,141 issues on `main`. 1,082 were sigma false-positives from `public/specs/portfolio.json` (a 280KB Polaris API spec) and `docs/archive/polaris-apis/*.yaml` (archived API specs). The 5 genuine high findings were `SIGMA.container_requesting_net_raw` in docker-compose files; the remaining ~54 were TypeScript SAST findings (`FORWARD_NULL`, `REVERSE_INULL`, `NO_EFFECT`, `DEADCODE`).

**What did not work**:
- `.sigma-config.yml` `analyze.exclude_file_path` — bypassed because Bridge passes `--coverity-config /tmp/...` to `cov-run-sigma`, which overrides sigma's auto-detection of its own config
- `project_source_excludes` in the GHA workflow — only applies to SOURCE_UPLOAD zip mode; no effect in hybrid/local capture mode
- `coverity.yml` `capture.files.exclude-glob: "docs/archive/**"` — `**` glob unsupported; AND it only accepts a single string value, not an array
- `coverity.yml` `capture.files.exclude-glob: "public/specs/*"` — `*` glob matched but `project_source_excludes` still not effective; needed to use `exclude-regex` instead

**What worked** (PR #70, commit 711e1fb):
```yaml
# coverity.yml
capture:
  files:
    exclude-regex: "(docs/archive/|public/specs/)"
```
Result: 1,141 → **59 issues** on the fix branch after one scan. All 59 are genuine findings in actual source files.
