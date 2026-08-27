# GitHub Setup Guide

Complete guide for configuring GitHub secrets, environments, and branch protection.

---

## GitHub Secrets Configuration

Navigate to: **Repository → Settings → Secrets and variables → Actions**

### Staging Environment Secrets

Create these repository secrets:

| Secret Name | Example Value | Description |
|-------------|---------------|-------------|
| `DATABRICKS_HOST_STAGING` | `https://adb-3150938829194941.1.azuredatabricks.net/` | Staging workspace URL |
| `DATABRICKS_CLIENT_ID_STAGING` | `1424fb7c-47c7-41a6-b845-e4d90ef4502e` | Service principal client ID |
| `DATABRICKS_CLIENT_SECRET_STAGING` | `<secret-value>` | Service principal secret |

### Production Environment Secrets

| Secret Name | Example Value | Description |
|-------------|---------------|-------------|
| `DATABRICKS_HOST` | `https://adb-2001924374254536.16.azuredatabricks.net/` | Production workspace URL |
| `DATABRICKS_CLIENT_ID` | `ef0d836c-c1d5-4d27-a310-dbab9baac7d8` | Service principal client ID |
| `DATABRICKS_CLIENT_SECRET` | `<secret-value>` | Service principal secret |

### Getting Service Principal Credentials

If you need to create a new service principal:

**Azure:**
```bash
az ad sp create-for-rbac \
  --name "databricks-cicd-staging" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<rg-name>
```

Output includes:
```json
{
  "clientId": "...",      # Use for DATABRICKS_CLIENT_ID
  "clientSecret": "...",  # Use for DATABRICKS_CLIENT_SECRET
  "tenantId": "..."
}
```

Then grant the service principal access to the Databricks workspace.

---

## GitHub Environments

Navigate to: **Repository → Settings → Environments**

### Create Staging Environment

1. Click "New environment"
2. Name: `staging` (lowercase, exact match)
3. **Protection rules:** Leave empty (allows automatic deployment)
4. **Deployment branches:** Select "Selected branches" → Add `main`
5. **Environment secrets:** Optional (repo secrets work fine)
6. **Reviewers:** Leave empty for staging
7. Save

### Create Production Environment

1. Click "New environment"
2. Name: `production` (lowercase, exact match)
3. **Protection rules:**
   - ✅ Enable "Required reviewers"
   - Add specific team members (e.g., senior engineers)
   - Set "Wait timer" to 0 minutes
   - ✅ Enable "Prevent self-review" (recommended)
4. **Deployment branches:** Select "Selected branches" → Add `main`
5. **Environment secrets:** Optional
6. Save

**Why reviewers matter:** This is your final gate before production changes. Only trusted team members should be reviewers.

---

## Branch Protection Rules

Navigate to: **Repository → Settings → Branches → Add rule**

### Protect Main Branch

| Setting | Configuration |
|---------|---------------|
| **Branch name pattern** | `main` |
| ✅ **Require pull request before merging** | Enabled |
| ✅ **Require approvals** | 1 minimum |
| ✅ **Require review from Code Owners** | Enabled (critical!) |
| ✅ **Dismiss stale reviews** | Enabled |
| ✅ **Require status checks to pass** | Enabled |
| **Status checks** | Add: `Staging Deployment & Validation` |
| ✅ **Require branches to be up to date** | Enabled |
| ✅ **Require conversation resolution** | Enabled (optional) |
| ✅ **Include administrators** | Enabled (recommended) |
| ✅ **Allow force pushes** | Disabled |
| ✅ **Allow deletions** | Disabled |

Click "Create" or "Save changes"

---

## CODEOWNERS File

Create `.github/CODEOWNERS` in your repository:

```
# Databricks Bundle CI/CD - Requires senior approval

# Individual pipelines
/data_manip_notebooks/superduck_dag_backlog/backlog_maintenance/ @cave @senior-engineer
/data_manip_notebooks/fastDAST_direct_ingress/fastdast_ingestion/ @cave @senior-engineer

# All Databricks orchestration
/data_manip_notebooks/**/databricks_orchestration/ @cave @senior-engineer

# CI/CD workflows
/.github/workflows/ @cave @lead-engineer

# Root-level bundle configs
/databricks*.yml @cave @senior-engineer
```

**Using GitHub Teams:**

If your organization has teams configured:

```
# Use @org/team-name format
/data_manip_notebooks/**/databricks_orchestration/ @your-org/senior-engineers
/.github/workflows/ @your-org/devops-team
```

**Path Pattern Tips:**
- No trailing slashes on directory paths
- Use `**` for recursive matching
- More specific rules override general ones
- Comments start with `#`

---

## Workflow File Checklist

When creating `.github/workflows/pipeline-cicd.yml`, verify:

- [ ] Workflow name is descriptive
- [ ] `on.pull_request.paths` matches bundle directory
- [ ] `on.push.paths` matches bundle directory  
- [ ] `env.BUNDLE_DIR` points to correct bundle path
- [ ] Staging job has `environment: name: staging`
- [ ] Production job has `environment: name: production`
- [ ] Production job has `if:` condition checking:
  - `github.ref == 'refs/heads/main'`
  - `github.event_name == 'push'`
  - `needs.staging.outputs.tests_passed == 'true'`
- [ ] All `databricks bundle` commands use `-t <target>` flag
- [ ] Profile names match environment (STAGING, PRODUCTION)
- [ ] Secrets are referenced with correct names

---

## Testing Your Setup

### Test 1: Verify Secrets

Create a test workflow that echoes (masked) secret presence:

```yaml
name: Test Secrets
on: workflow_dispatch
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Check Secrets
        env:
          STAGING_HOST: ${{ secrets.DATABRICKS_HOST_STAGING }}
          PROD_HOST: ${{ secrets.DATABRICKS_HOST }}
        run: |
          if [ -n "$STAGING_HOST" ]; then
            echo "✅ Staging host configured"
          else
            echo "❌ Staging host missing"
          fi
          if [ -n "$PROD_HOST" ]; then
            echo "✅ Production host configured"
          else
            echo "❌ Production host missing"
          fi
```

### Test 2: Verify Branch Protection

1. Create a test branch
2. Make a small change
3. Push and create PR
4. Verify:
   - Code owner is automatically requested
   - Cannot merge without approval
   - Status checks appear and must pass

### Test 3: Verify Environment Protection

1. Manually trigger a workflow with production deployment
2. Verify it pauses waiting for approval
3. Check that only designated reviewers can approve
4. Approve and verify deployment proceeds

### Test 4: End-to-End Pipeline

1. Create feature branch with small change
2. Push and create PR
3. Wait for CODEOWNERS approval
4. Verify staging deployment runs
5. Merge PR
6. Verify production waits for approval
7. Approve and verify production deployment
8. Check Databricks UI for deployed resources

---

## Troubleshooting Setup Issues

### "Required status check does not exist"

**Problem:** Branch protection requires a status check that hasn't run yet.

**Fix:** 
1. Temporarily disable the status check requirement
2. Merge one PR to establish the check
3. Re-enable the requirement

### "Code Owners rules not applied"

**Problem:** PR doesn't request code owner review.

**Fixes:**
- Verify file is `.github/CODEOWNERS` (not `CODEOWNERS` in root)
- Check path patterns match exactly
- Ensure branch protection has "Require review from Code Owners" enabled
- Verify code owners are valid GitHub usernames/teams

### "Deployment never asks for approval"

**Problem:** Production deploys without manual approval.

**Fixes:**
- Verify environment name is exactly `production` in workflow
- Check environment has "Required reviewers" configured
- Ensure reviewers list is not empty
- Verify workflow job specifies `environment: name: production`

### "Secrets not found in Actions"

**Problem:** Workflow fails with authentication errors.

**Fixes:**
- Verify secrets created under Actions (not Dependabot or Codespaces)
- Check secret names match exactly (case-sensitive)
- Ensure secrets are at repository level, not environment level (unless intended)
- For private repos, verify Actions are enabled

---

## Security Best Practices

1. **Rotate service principal secrets regularly** - Every 90 days minimum
2. **Use separate service principals** - Don't reuse staging credentials in prod
3. **Limit service principal permissions** - Only grant what's needed
4. **Review environment reviewers** - Remove users who leave the team
5. **Audit secret access** - Check GitHub's audit log periodically
6. **Use environment secrets for sensitive data** - More restrictive than repo secrets
7. **Enable Dependabot alerts** - For workflow dependencies
8. **Require signed commits** - For extra security (optional)

---

## Maintenance

### Quarterly Tasks

- Review and rotate service principal secrets
- Audit CODEOWNERS list - update for team changes
- Review environment protection reviewers
- Check for deprecated GitHub Actions versions in workflows

### When Team Members Leave

- Remove from CODEOWNERS
- Remove from environment reviewers
- Rotate any secrets they had access to
- Review recent PRs they approved

### When Workspaces Change

- Update `DATABRICKS_HOST` secrets
- Update workspace URLs in environment configuration
- Update cluster IDs in bundle databricks.yml
- Test deployments in both environments
