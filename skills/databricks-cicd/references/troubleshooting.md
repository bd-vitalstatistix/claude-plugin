# Troubleshooting Databricks CI/CD

Common issues and how to fix them.

---

## Environment Always Shows Staging in Production

**Symptom:** Job runs in prod workspace but uses staging catalogs/clusters.

**Root Cause:** The `environment` parameter isn't being passed from the GitHub workflow to the notebook.

**The Connection Chain:**
```
GitHub workflow: -t prod
    ↓
databricks.yml: ${bundle.target} = "prod"
    ↓
job.yml: environment: "${bundle.target}"
    ↓
Notebook: dbutils.widgets.get("environment") = "prod"
```

**Check These Places:**

### 1. Job Definition Missing Environment Parameter

**Problem:** The most common issue is forgetting `environment` in for-each tasks.

Look in `resources/job_name.job.yml`:

```yaml
for_each_task:
  task:
    notebook_task:
      base_parameters:
        input: "{{input}}"
        job_id: "${var.job_id_param}"
        # Missing: environment: "${bundle.target}"
```

**Fix:** Add it to every task, including iterations:

```yaml
for_each_task:
  task:
    notebook_task:
      base_parameters:
        input: "{{input}}"
        job_id: "${var.job_id_param}"
        environment: "${bundle.target}"  # Add this!
```

### 2. Workflow Missing Target Flag

Check `.github/workflows/pipeline-cicd.yml`:

```yaml
# Wrong - no target specified
run: databricks bundle deploy --profile PRODUCTION

# Correct
run: databricks bundle deploy -t prod --profile PRODUCTION
```

### 3. Notebook Hardcodes Environment

Check the notebook:

```python
# Wrong - always staging
environment = "staging"

# Correct - reads from parameter
try:
    environment = dbutils.widgets.get("environment")
except:
    environment = "staging"
```

### 4. Forgot to Redeploy

Changes to job.yml don't take effect until you redeploy:

```bash
databricks bundle deploy -t prod --profile PRODUCTION --force-lock
```

**Debugging Tip:** Check the actual job parameters in Databricks UI:
1. Go to the job run in Databricks
2. Click on the task
3. Look at "Parameters" tab
4. Does it show `environment: prod` or `environment: staging`?

---

## Authentication Failures

**Symptom:** GitHub Actions fails with "authentication error" or "secrets not found".

**Fixes:**

### 1. Verify Secrets Exist

Go to GitHub repo → Settings → Secrets and variables → Actions

Required secrets:
- `DATABRICKS_HOST_STAGING`
- `DATABRICKS_CLIENT_ID_STAGING`
- `DATABRICKS_CLIENT_SECRET_STAGING`
- `DATABRICKS_HOST` (prod)
- `DATABRICKS_CLIENT_ID` (prod)
- `DATABRICKS_CLIENT_SECRET` (prod)

Secret names are case-sensitive and must match exactly.

### 2. Check Environment Configuration

If using GitHub Environments:
- Environment names must match: `staging`, `production` (lowercase)
- The workflow must specify: `environment: name: staging`

### 3. Service Principal Permissions

The service principal needs:
- Workspace access
- CAN_MANAGE permissions on the bundle
- Cluster access (if using existing clusters)

---

## Job Validation Errors

**Symptom:** `databricks bundle validate` fails.

**Common Causes:**

### 1. Missing Notebooks

```
Error: notebook not found: ./notebook.ipynb
```

**Fix:** Check `sync.include` paths in databricks.yml match actual file locations:

```yaml
sync:
  include:
    - /resources/notebook.ipynb  # Relative to bundle root
```

### 2. Invalid Cluster ID

```
Error: cluster 'xxx' does not exist
```

**Fix:** Update `cluster_id` variable in databricks.yml for correct workspace:

```yaml
targets:
  staging:
    variables:
      cluster_id: "0506-121432-nni9vpnm"  # Check in Databricks UI
```

### 3. Syntax Errors in YAML

```
Error: yaml: line 45: could not find expected ':'
```

**Fix:** YAML is whitespace-sensitive. Common issues:
- Mixed tabs and spaces (use spaces only)
- Incorrect indentation
- Missing colons after keys
- Unquoted strings with special characters

Use a YAML validator or VS Code YAML extension.

---

## CODEOWNERS Not Working

**Symptom:** PR doesn't request review from code owners.

**Fixes:**

### 1. File Location

Must be at `.github/CODEOWNERS` (not in repo root).

### 2. Branch Protection

Enable in Settings → Branches → Branch protection rules → "Require review from Code Owners"

### 3. Path Patterns

```
# Wrong - trailing slash
/path/to/bundle/ @reviewer

# Correct - no trailing slash, or use **
/path/to/bundle/** @reviewer
```

### 4. User/Team Names

```
# User must be GitHub username with @
@username

# Team must be org/team format
@your-org/team-name
```

---

## Production Deployment Doesn't Require Approval

**Symptom:** Production deploys without manual approval.

**Fixes:**

### 1. Create Production Environment

Settings → Environments → New environment → `production`

### 2. Add Required Reviewers

In production environment settings:
- Enable "Required reviewers"
- Add specific users or teams
- Save protection rules

### 3. Workflow Must Specify Environment

```yaml
production:
  name: Production Deployment
  environment:  # This line is critical
    name: production
```

---

## Staging Tests Pass But Production Doesn't Deploy

**Symptom:** Staging succeeds, but production job never starts.

**Check These Conditions:**

### 1. Branch Condition

```yaml
if: |
  github.ref == 'refs/heads/main' &&  # Only on main branch
  github.event_name == 'push' &&
  needs.staging.outputs.tests_passed == 'true'
```

If you merged via GitHub UI, event is `push`. If you rebased or force-pushed, might not trigger.

### 2. Staging Outputs

The staging job must set `tests_passed` output:

```yaml
staging:
  outputs:
    tests_passed: ${{ steps.check_tests.outputs.passed }}
  steps:
    - name: Check Job Results
      id: check_tests
      run: |
        if [ "${{ steps.monitor_job.outputs.result }}" = "success" ]; then
          echo "passed=true" >> $GITHUB_OUTPUT
        fi
```

---

## Bundle Deployment Conflicts

**Symptom:** "deployment already in progress" or "failed to acquire lock".

**Cause:** Another deployment is running, or previous deployment didn't clean up.

**Fixes:**

### 1. Wait for Other Deployment

Check GitHub Actions or ask team members if someone else is deploying.

### 2. Force Lock

```bash
databricks bundle deploy -t <target> --force-lock
```

Use sparingly - can interrupt active deployments.

### 3. Check Databricks UI

Sometimes a job is stuck. Go to Databricks → Workflows → Find the job → Cancel any stuck runs.

---

## Notebook Not Found After Deployment

**Symptom:** Job deployed but fails with "notebook not found".

**Causes:**

### 1. Sync Path Issues

The notebook path in job.yml must match what's synced:

```yaml
# In databricks.yml
sync:
  include:
    - /resources/my_notebook.ipynb

# In job.yml - must match
notebook_task:
  notebook_path: ./my_notebook.ipynb  # No /resources/ prefix!
```

### 2. Workspace Path vs Bundle Path

Job uses bundle-relative paths (with `./`), not absolute workspace paths.

**Wrong:**
```yaml
notebook_path: /Workspace/Shared/.bundle/my-bundle/prod/my_notebook.ipynb
```

**Correct:**
```yaml
notebook_path: ./my_notebook.ipynb
```

---

## High Token/Time Usage in Production

**Symptom:** Production jobs take much longer or cost more than staging.

**Possible Causes:**

### 1. Different Cluster Sizes

Check `cluster_id` variables - prod and staging should use appropriate cluster sizes.

### 2. Data Volume Differences

Production might process significantly more data. Consider:
- Adding indexes
- Optimizing queries
- Increasing cluster size
- Using job clusters instead of all-purpose clusters

### 3. Concurrent Runs

Check `jobs_max_concurrent_runs` in databricks.yml. If too high, might be running unnecessary parallel jobs.

---

## Notebook Not Found (Double Path Error)

**Symptom:** `Error: notebook resources/resources/notebook.ipynb not found`

**Cause:** Sync paths and job paths mismatch, creating double `resources/` prefix.

**Fix:** In `resources/job.yml`, use `./notebook.ipynb` NOT `./resources/notebook.ipynb`:

```yaml
# Wrong
notebook_path: ./resources/postgres_backup.ipynb  # Creates resources/resources/

# Correct
notebook_path: ./postgres_backup.ipynb
```

Sync config already puts notebooks at bundle root.

---

## Secrets Permission Denied

**Symptom:** `Permission denied` when accessing `dbutils.secrets.get()`

**Cause:** Service principal lacks READ access to secrets scope.

**Fix:** Grant READ permission to both SPs:

```bash
# Staging
databricks secrets put-acl <scope-name> <staging-sp-id> READ --profile STAGING

# Production
databricks secrets put-acl <scope-name> <prod-sp-id> READ --profile PRODUCTION
```

---

## Secrets Scope Not Found

**Symptom:** `Scope '<scope>' does not exist`

**Cause:** Secrets scopes are workspace-specific, not shared between staging/prod.

**Fix:** Create scope in BOTH workspaces with identical secrets:

```bash
# Create in staging
databricks secrets create-scope <scope-name> --profile STAGING
databricks secrets put-secret <scope-name> <key-name> --profile STAGING

# Create in production
databricks secrets create-scope <scope-name> --profile PRODUCTION
databricks secrets put-secret <scope-name> <key-name> --profile PRODUCTION
```

---

## Tips for Effective Debugging

1. **Check Databricks UI first** - Often shows more detailed errors than GitHub logs
2. **Compare staging vs prod** - Look at actual deployed job definitions in Databricks
3. **Use `--verbose` flag** - `databricks bundle deploy -t prod --verbose`
4. **Check service principal logs** - In Azure/AWS console
5. **Test locally first** - `databricks bundle validate -t <target>` before pushing
6. **Read the full GitHub Actions log** - Don't just look at the summary, expand all steps
