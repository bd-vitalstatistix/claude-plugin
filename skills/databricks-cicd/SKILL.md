---
name: databricks-cicd
description: Create GitHub Actions CI/CD pipelines for Databricks Asset Bundles with staging-to-production workflows. USE THIS SKILL whenever the user mentions databricks deployment, CI/CD for databricks, github actions workflows, staging to production pipelines, databricks bundle deployment, automated databricks workflows, or environment configuration issues (especially when production shows staging catalogs/clusters). Also use for troubleshooting databricks CI/CD, setting up approval gates, configuring GitHub environments, or creating new databricks pipelines with automated testing. Even if the user doesn't explicitly say "CI/CD", use this skill when they want to automate databricks deployments, set up workflows, or need environment separation between staging and production.
---

# Databricks Asset Bundle CI/CD

Automate Databricks Asset Bundle deployments through GitHub Actions with staging validation and production approval gates.

## When to Use This Skill

- Creating new CI/CD pipelines for Databricks bundles
- Setting up automated staging → production workflows  
- Troubleshooting environment issues (prod using staging resources)
- Configuring GitHub approval gates and branch protection
- Converting manual deployments to automated pipelines

---

## Understanding the Environment Flow

The most common issue in Databricks CI/CD is production jobs using staging resources. This happens because of a broken parameter chain. Here's how it should work:

```
GitHub workflow specifies target → Databricks resolves variables → Notebook receives environment
      databricks deploy -t prod            ${bundle.target} = "prod"         widget.get("environment")
```

Each link in this chain must be correct:

1. **GitHub workflow** uses `-t staging` or `-t prod` flag
2. **databricks.yml** defines target-specific variables  
3. **job.yml** passes `environment: "${bundle.target}"` to notebooks
4. **Notebook** reads from widget, doesn't hardcode

Missing any link causes notebooks to fall back to the default (usually "staging").

---

## Quick Start: Creating a New Pipeline

When the user asks to create a new Databricks CI/CD pipeline:

### 1. Understand the Requirements

Ask about:
- Pipeline name and purpose
- Notebooks to include
- Schedule (if any)
- **Email notifications for failures** - If not provided, MUST prompt user for email addresses to receive alerts when jobs fail
- Special dependencies or requirements

**IMPORTANT:** Always collect email addresses for failure notifications. If the user doesn't provide any, explicitly ask: "Which email address(es) should receive alerts when this job fails?" This is critical for operational visibility.

### 2. Create Bundle Structure

```
pipeline_name/
├── databricks.yml           # Bundle configuration
├── resources/
│   └── job_name.job.yml    # Job definition
└── resources/              
    └── notebook.ipynb       # Processing notebooks
```

### 3. Generate Configuration Files

Use the complete templates in `references/templates.md`:

- **databricks.yml** - Defines staging and prod targets with workspace URLs, service principals, cluster IDs
- **job.yml** - Defines tasks with environment parameters and email_notifications section with on_failure addresses
- **workflow.yml** - GitHub Actions for automated deployment
- **README.md** - Developer documentation

**Critical details:** 
- Every task in job.yml needs `environment: "${bundle.target}"` in base_parameters (includes for-each task iterations)
- The job.yml MUST include email_notifications with at least one email address for failure alerts

### 4. Set Up GitHub

Follow `references/github-setup.md` for:

- Configuring GitHub secrets (workspace URLs, service principal credentials)
- Creating environments (staging with no protection, production with required reviewers)
- Setting up branch protection (require PR reviews, status checks)
- Adding CODEOWNERS (enforce senior approval)

### 5. Test the Pipeline

1. Create a feature branch with the new bundle
2. Push and create PR (triggers staging deployment)
3. Verify staging deployment succeeds
4. Merge to main (triggers production approval gate)
5. Approve production deployment
6. Verify production deployment succeeds

---

## The Environment Parameter Pattern

This is the single most important pattern to get right. Without it, production will use staging resources.

### In databricks.yml

Define environment variable per target:

```yaml
variables:
  environment:
    default: staging

targets:
  staging:
    variables:
      environment: staging
      admin_catalog: dataagg_admin_staging
  
  prod:
    variables:
      environment: prod
      admin_catalog: dataagg_admin
```

### In job.yml

Pass environment to every task:

```yaml
tasks:
  - task_key: process_data
    notebook_task:
      notebook_path: ./notebook.ipynb
      base_parameters:
        environment: "${bundle.target}"  # Resolves to staging or prod
```

**For for-each tasks**, add it to the iteration:

```yaml
for_each_task:
  inputs: "{{tasks.discovery.values.dataset_list}}"
  task:
    notebook_task:
      base_parameters:
        input: "{{input}}"
        environment: "${bundle.target}"  # Don't forget this!
```

The `${bundle.target}` variable automatically matches whatever `-t` flag was used in the GitHub workflow (`-t staging` → `"staging"`, `-t prod` → `"prod"`).

### In Notebooks

Read from widget, don't hardcode:

```python
try:
    dbutils.widgets.text("environment", "staging", "Environment")
    environment = dbutils.widgets.get("environment")
except:
    environment = "staging"  # Fallback if not provided

# Use environment to select resources
if environment == "staging":
    catalog = "silver_staging"
    admin_catalog = "dataagg_admin_staging"
else:
    catalog = "silver"
    admin_catalog = "dataagg_admin"
```

This reads the parameter passed from job.yml. Without the widget, the notebook won't receive the environment value.

### In GitHub Workflows

Use the `-t` flag to specify the target:

```bash
# Staging deployment
databricks bundle deploy -t staging --profile STAGING

# Production deployment  
databricks bundle deploy -t prod --profile PRODUCTION
```

The `-t` flag sets the target, which flows through the entire chain.

---

## Why This Matters

When any link is broken, notebooks receive no environment parameter and fall back to their default (usually hardcoded as "staging"). This causes:

- Production jobs reading from staging catalogs
- Production jobs writing to staging tables
- Production jobs using staging clusters (wrong size/config)
- Data corruption or mixing of staging/prod data

The symptom is always the same: "I deployed to prod but it's still using staging resources."

The fix is always the same: Ensure `environment: "${bundle.target}"` is in every task's base_parameters, then redeploy.

---

## Common Scenarios

### Scenario 1: New Pipeline from Scratch

1. Read `references/templates.md` for complete file templates
2. Create bundle directory structure
3. Copy and customize templates (databricks.yml, job.yml, workflow.yml)
4. Set up GitHub per `references/github-setup.md`
5. Test with a PR to staging

### Scenario 2: Existing Pipeline Missing CI/CD

1. Verify bundle structure is correct
2. Read `references/templates.md` for workflow template
3. Create `.github/workflows/pipeline-name-cicd.yml`
4. Set up GitHub environments and secrets
5. Add bundle path to `.github/CODEOWNERS`
6. Test with a PR

**Note**: If the notebooks are located outside the bundle directory, copy them into the bundle structure and leave the originals unedited. The bundle needs its own copy of the notebooks to manage deployments independently. Update the `sync.include` paths in databricks.yml to point to the bundle's notebook copies.

### Scenario 3: Production Using Staging Resources

This is an environment parameter issue. Follow `references/troubleshooting.md` section "Environment Always Shows Staging in Production" for diagnosis.

Quick check:
1. Open job.yml - does every task have `environment: "${bundle.target}"`?
2. Check for-each tasks - they're commonly missed
3. After fixing, redeploy: `databricks bundle deploy -t prod --force-lock`

### Scenario 4: Authentication Failures

See `references/github-setup.md` for complete secret configuration or `references/troubleshooting.md` for debugging authentication issues.

### Scenario 5: Approval Gate Not Working

See `references/troubleshooting.md` section "Production Deployment Doesn't Require Approval" for environment protection configuration.

---

## File Organization

This skill includes reference documentation for common tasks:

### references/templates.md

Complete copy-paste templates for:
- databricks.yml with staging/prod targets
- job.yml with simple and for-each tasks
- GitHub Actions workflow
- Bundle README.md
- Notebook environment widget pattern
- Staging data setup with shallow clones (for pipelines that depend on shared prod tables)

Use these as starting points. All templates include proper environment parameter configuration.

**Note on shallow clones**: If your notebooks depend on tables that exist in production but not in staging (like admin tables, lookup tables, or processing backlogs), see the "Staging Data Setup: Shallow Clone Pattern" in templates.md. This pattern lets staging work with realistic prod data snapshots without manual setup.

### references/troubleshooting.md

Solutions for common issues:
- Environment always shows staging in production
- Authentication failures
- Job validation errors
- CODEOWNERS not working
- Production deployment doesn't require approval
- Bundle deployment conflicts
- Notebook not found errors
- High token/time usage

Read this when debugging issues. It includes the root causes and step-by-step fixes.

### references/github-setup.md

Complete guide for GitHub configuration:
- Secrets setup (staging and production)
- Environment creation and protection rules
- Branch protection rules
- CODEOWNERS file format
- Testing your setup
- Security best practices
- Maintenance procedures

Follow this when setting up GitHub for a new pipeline or troubleshooting GitHub-specific issues.

---

## Workflow Pattern

All pipelines follow this pattern:

```
Developer creates PR
        ↓
Code review required (CODEOWNERS)
        ↓
Staging deployment runs automatically
        ↓
Tests/validation execute
        ↓
PR merged to main
        ↓
Staging runs again
        ↓
Production waits for approval
        ↓
Senior approves
        ↓
Production deployment
        ↓
Tag created
```

### Key Points

**Two approval gates:**
1. **Code review** - Before merge, enforced by CODEOWNERS and branch protection
2. **Production deployment** - After staging tests pass, enforced by GitHub Environment

**Staging always runs first.** Even on main branch, staging validates before production is triggered.

**Production requires approval.** Configure this in GitHub Environment settings with required reviewers.

**Automatic tagging.** Each production deployment creates a git tag for rollback capability.

---

## Validation Checklist

Before deploying a new pipeline, verify:

**Bundle Configuration:**
- [ ] databricks.yml exists with staging and prod targets
- [ ] Each target defines environment, admin_catalog, cluster_id variables
- [ ] Sync paths point to actual notebook locations

**Job Definition:**
- [ ] Every task has `environment: "${bundle.target}"` in base_parameters
- [ ] For-each tasks also include environment parameter
- [ ] Email notifications are configured with at least one email address for on_failure alerts
- [ ] Cluster IDs are correct for each environment

**GitHub Configuration:**
- [ ] Workflow file exists in `.github/workflows/`
- [ ] Workflow uses `-t staging` and `-t prod` flags correctly
- [ ] GitHub secrets are configured (6 total: 3 staging, 3 prod)
- [ ] Staging environment exists (no protection)
- [ ] Production environment exists (required reviewers configured)
- [ ] Branch protection enabled on main (require PR + reviews)
- [ ] Bundle path added to `.github/CODEOWNERS`

**Notebooks:**
- [ ] Read environment from dbutils.widgets.get()
- [ ] Don't hardcode environment = "staging"
- [ ] Use environment to select catalogs/clusters
- [ ] If dependent on shared prod tables, implement shallow clone setup in first task (see templates.md)

**Testing:**
- [ ] Can deploy to staging manually
- [ ] Can run staging job successfully
- [ ] PR triggers staging deployment
- [ ] Production requires approval

---

## Quick Commands

```bash
# Validate bundle configuration
databricks bundle validate -t staging

# Deploy to staging
databricks bundle deploy -t staging --profile STAGING --force-lock

# Deploy to production
databricks bundle deploy -t prod --profile PRODUCTION --force-lock

# Run job manually
databricks bundle run job_name -t staging --profile STAGING

# Destroy deployment
databricks bundle destroy -t staging --profile STAGING
```

---

## Best Practices

### Explain Why, Don't Just Mandate

When writing job definitions or documentation, explain why the environment parameter matters rather than just marking it as required. This helps developers understand the flow and catch mistakes.

### Test Locally First

Run `databricks bundle validate` locally before pushing. Catches syntax errors and missing notebooks early.

### Use Descriptive Names

- Bundle names: `pipeline_name` not `new_thing`
- Task keys: `discover_datasets` not `task1`
- Job names: `process_dag_backlog_${bundle.target}` not `job`

### Keep Bundles Independent

Each bundle should be completely independent with its own databricks.yml and deployment lifecycle. Don't share configurations across bundles.

### Document in README

Every bundle needs a README.md explaining:
- What the pipeline does
- How to deploy it
- Common issues and solutions
- Contact information

### Monitor Token Usage

Production jobs might process more data than staging. Watch for unexpectedly high token counts or long runtimes and optimize accordingly.

### Pin Package Versions

**CRITICAL:** All Python packages in job definitions MUST be pinned to specific versions for reproducibility and security.

- **Prefer SHA-256 hash pinning** for maximum security and reproducibility
- **Minimum requirement:** Pin to exact release version (e.g., `psycopg[binary]==3.1.8`)
- **Never use unpinned packages** (e.g., `psycopg` or `psycopg>=3.0`)

Example in job.yml:
```yaml
libraries:
  - pypi:
      package: "psycopg[binary]==3.1.8"  # Pinned to exact version
```

This prevents:
- Breaking changes from automatic upgrades
- Inconsistent behavior between staging and production
- Security vulnerabilities from unvetted versions

---

## Progressive Disclosure

This SKILL.md provides an overview and key patterns. For detailed information:

- **Complete templates** → `references/templates.md` (400+ lines of copy-paste ready code)
- **Troubleshooting** → `references/troubleshooting.md` (detailed solutions for common issues)
- **GitHub setup** → `references/github-setup.md` (complete configuration guide)

Load these reference files when you need specific details or are helping debug an issue.

---

## Related Resources

- [Databricks Asset Bundles Documentation](https://docs.databricks.com/dev-tools/bundles/)
- [GitHub Actions Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [GitHub CODEOWNERS Syntax](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
