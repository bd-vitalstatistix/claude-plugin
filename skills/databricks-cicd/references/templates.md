# Complete Templates for Databricks CI/CD

This file contains full, copy-paste ready templates for all files needed in a Databricks CI/CD pipeline.

---

## databricks.yml Template

```yaml
# Databricks Asset Bundle - [Pipeline Name]
bundle:
  name: pipeline_name

include:
  - resources/*.yml

sync:
  include:
    - /resources/notebook1.ipynb
    - /resources/notebook2.ipynb

variables:
  environment:
    description: "Deployment environment (staging/prod)"
    default: staging
  
  admin_catalog:
    description: "Admin catalog for data tables"
    default: dataagg_admin_staging
  
  cluster_id:
    description: "Cluster ID for job execution"
    default: "0506-121432-nni9vpnm"
  
  job_id_param:
    description: "Job ID parameter passed to tasks"
    default: "{{job.id}}"
  
  run_id_param:
    description: "Run ID parameter passed to tasks"
    default: "{{job.run_id}}"

targets:
  staging:
    workspace:
      root_path: /Workspace/Shared/.bundle/${bundle.name}/${bundle.target}
      host: https://adb-3150938829194941.1.azuredatabricks.net/
    presets:
      name_prefix: "[staging] "
      trigger_pause_status: PAUSED
      pipelines_development: false
      jobs_max_concurrent_runs: 1
      tags:
        environment: staging
    permissions:
      - level: CAN_MANAGE
        service_principal_name: "1f9742ae-474f-4efa-899e-3f408d5006e0"
      - level: CAN_MANAGE
        group_name: users
    variables:
      environment: staging
      admin_catalog: dataagg_admin_staging
      cluster_id: "0506-121432-nni9vpnm"
    run_as:
      service_principal_name: "1424fb7c-47c7-41a6-b845-e4d90ef4502e"

  prod:
    mode: production
    workspace:
      root_path: /Workspace/Shared/.bundle/${bundle.name}/${bundle.target}
      host: https://adb-2001924374254536.16.azuredatabricks.net/
    presets:
      trigger_pause_status: UNPAUSED
      pipelines_development: false
      jobs_max_concurrent_runs: 1
      tags:
        environment: prod
    permissions:
      - level: CAN_MANAGE
        service_principal_name: "f4c16c07-e707-4ede-a779-61983784240d"
      - level: CAN_MANAGE
        group_name: users
    variables:
      environment: prod
      admin_catalog: dataagg_admin
      cluster_id: "0723-124438-stprs4gy"
    run_as:
      service_principal_name: "51e1a37d-762b-49c4-b198-d604e74ac1f1"
```

---

## Job Definition Template

```yaml
resources:
  jobs:
    job_name:
      name: job_name_${bundle.target}
      
      description: >-
        Job description here

      schedule:
        quartz_cron_expression: "0 0 0 ? * MON *"
        timezone_id: "UTC"
        pause_status: PAUSED

      email_notifications:
        on_failure:
          - team@company.com

      timeout_seconds: 7200

      tasks:
        # Simple task example
        - task_key: simple_task
          existing_cluster_id: "${var.cluster_id}"
          notebook_task:
            notebook_path: ./notebook.ipynb
            source: WORKSPACE
            base_parameters:
              job_id: "${var.job_id_param}"
              run_id: "${var.run_id_param}"
              environment: "${bundle.target}"  # Links workflow -t flag to notebook

        # For-each task example
        - task_key: parallel_task
          depends_on:
            - task_key: simple_task
          for_each_task:
            inputs: "{{tasks.simple_task.values.dataset_list}}"
            concurrency: 4
            task:
              task_key: parallel_task_iteration
              existing_cluster_id: "${var.cluster_id}"
              notebook_task:
                notebook_path: ./processing_notebook.ipynb
                source: WORKSPACE
                base_parameters:
                  input: "{{input}}"
                  job_id: "${var.job_id_param}"
                  run_id: "${var.run_id_param}"
                  environment: "${bundle.target}"  # Don't forget in iterations!
              max_retries: 2
              min_retry_interval_millis: 120000

      tags:
        pipeline: "pipeline_name"
        owner: "owner_name"
        environment: "${bundle.target}"
```

---

## GitHub Actions Workflow Template

```yaml
name: Pipeline CI/CD

on:
  pull_request:
    paths:
      - 'path/to/bundle/**'
      - '.github/workflows/pipeline-cicd.yml'
  
  push:
    branches:
      - main
    paths:
      - 'path/to/bundle/**'
      - '.github/workflows/pipeline-cicd.yml'

  workflow_dispatch:
    inputs:
      skip_staging:
        description: 'Skip staging deployment (emergency prod-only deploy)'
        required: false
        type: boolean
        default: false

env:
  BUNDLE_DIR: path/to/bundle

jobs:
  staging:
    name: Staging Deployment & Validation
    runs-on: ubuntu-latest
    if: github.event.inputs.skip_staging != 'true'
    environment: 
      name: staging
      url: https://adb-3150938829194941.1.azuredatabricks.net/
    
    outputs:
      tests_passed: ${{ steps.check_tests.outputs.passed }}
      staging_run_id: ${{ steps.run_job.outputs.run_id }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Databricks CLI
        run: |
          curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
          databricks --version
      
      - name: Configure Databricks Profile
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_STAGING }}
          DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_CLIENT_ID_STAGING }}
          DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_CLIENT_SECRET_STAGING }}
        run: |
          mkdir -p ~/.databrickscfg.d
          cat > ~/.databrickscfg << EOF
          [STAGING]
          host = $DATABRICKS_HOST
          client_id = $DATABRICKS_CLIENT_ID
          client_secret = $DATABRICKS_CLIENT_SECRET
          auth_type = oauth-m2m
          EOF
          echo "✅ Databricks profile 'STAGING' configured"

      - name: Validate Bundle Configuration
        working-directory: ${{ env.BUNDLE_DIR }}
        run: databricks bundle validate -t staging --profile STAGING

      - name: Deploy to Staging
        working-directory: ${{ env.BUNDLE_DIR }}
        run: databricks bundle deploy -t staging --profile STAGING --force-lock

      - name: Run Staging Job
        id: run_job
        working-directory: ${{ env.BUNDLE_DIR }}
        run: |
          echo "🚀 Starting job in staging..."
          OUTPUT=$(databricks bundle run job_name -t staging --profile STAGING 2>&1 || true)
          echo "$OUTPUT"
          RUN_ID=$(echo "$OUTPUT" | grep -oE '/runs?/[0-9]+' | grep -oE '[0-9]+' | head -1)
          if [ -z "$RUN_ID" ]; then
            echo "❌ Failed to start job"
            exit 1
          fi
          echo "run_id=$RUN_ID" >> $GITHUB_OUTPUT

      - name: Monitor Job Execution
        id: monitor_job
        run: |
          RUN_ID="${{ steps.run_job.outputs.run_id }}"
          MAX_WAIT=1800
          ELAPSED=0
          INTERVAL=30
          
          while [ $ELAPSED -lt $MAX_WAIT ]; do
            RESPONSE=$(databricks jobs get-run $RUN_ID --profile STAGING --output json)
            STATUS=$(echo "$RESPONSE" | jq -r '.state.life_cycle_state')
            RESULT=$(echo "$RESPONSE" | jq -r '.state.result_state // "RUNNING"')
            
            if [ "$STATUS" = "TERMINATED" ]; then
              if [ "$RESULT" = "SUCCESS" ]; then
                echo "✅ Job completed successfully"
                echo "result=success" >> $GITHUB_OUTPUT
                exit 0
              else
                echo "❌ Job failed"
                echo "result=failed" >> $GITHUB_OUTPUT
                exit 1
              fi
            fi
            
            sleep $INTERVAL
            ELAPSED=$((ELAPSED + INTERVAL))
          done

      - name: Check Job Results
        id: check_tests
        if: always()
        run: |
          if [ "${{ steps.monitor_job.outputs.result }}" = "success" ]; then
            echo "passed=true" >> $GITHUB_OUTPUT
          else
            echo "passed=false" >> $GITHUB_OUTPUT
          fi

  production:
    name: Production Deployment
    runs-on: ubuntu-latest
    needs: staging
    if: |
      github.ref == 'refs/heads/main' && 
      github.event_name == 'push' &&
      needs.staging.outputs.tests_passed == 'true'
    environment: 
      name: production
      url: https://adb-2001924374254536.16.azuredatabricks.net/
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Databricks CLI
        run: |
          curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

      - name: Configure Databricks Profile
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_CLIENT_ID }}
          DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_CLIENT_SECRET }}
        run: |
          mkdir -p ~/.databrickscfg.d
          cat > ~/.databrickscfg << EOF
          [PRODUCTION]
          host = $DATABRICKS_HOST
          client_id = $DATABRICKS_CLIENT_ID
          client_secret = $DATABRICKS_CLIENT_SECRET
          auth_type = oauth-m2m
          EOF

      - name: Validate Bundle Configuration
        working-directory: ${{ env.BUNDLE_DIR }}
        run: databricks bundle validate -t prod --profile PRODUCTION

      - name: Deploy to Production
        working-directory: ${{ env.BUNDLE_DIR }}
        run: databricks bundle deploy -t prod --profile PRODUCTION --force-lock

      - name: Create Deployment Tag
        run: |
          TAG="pipeline-prod-$(date +%Y%m%d-%H%M%S)"
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git tag -a "$TAG" -m "Production deployment: ${{ github.sha }}"
          git push origin "$TAG"
```

---

## README.md Template for Bundle

```markdown
# [Pipeline Name] Bundle

**Independent Databricks Asset Bundle** for [pipeline description].

## Overview

[What this pipeline does]

## Quick Start

\`\`\`bash
cd path/to/bundle

# Validate
databricks bundle validate -t staging

# Deploy to staging
databricks bundle deploy -t staging

# Run manually
databricks bundle run job_name -t staging
\`\`\`

## Deployment Targets

| Target | Mode | Schedule | Description |
|--------|------|----------|-------------|
| `staging` | presets | Paused | Shared pre-production |
| `prod` | production | Active | Production environment |

## CI/CD Pipeline

This bundle uses automated GitHub Actions deployment:

\`\`\`
PR created → Senior review → Merge to main
    ↓              ↓              ↓
Staging deploy → Tests pass → Prod approval → Prod deploy
\`\`\`

### For Developers

1. Create feature branch
2. Make changes
3. Push and create PR
4. Wait for senior approval
5. Merge PR
6. Production deployment requires final approval

Production deployments should only go through CI/CD pipeline.

## Common Commands

\`\`\`bash
# Validate
databricks bundle validate -t <target>

# Deploy
databricks bundle deploy -t <target> --force-lock

# Run job
databricks bundle run job_name -t <target>

# Destroy
databricks bundle destroy -t <target>
\`\`\`

## Monitoring

- **Staging:** https://adb-3150938829194941.1.azuredatabricks.net/#job/
- **Production:** https://adb-2001924374254536.16.azuredatabricks.net/#job/
```

---

## Notebook Environment Widget Pattern

Notebooks should read the environment from parameters, not hardcode it:

```python
# Cell 1: Environment setup
try:
    dbutils.widgets.text("environment", "staging", "Environment")
    environment = dbutils.widgets.get("environment")
    if not environment:  # Handle empty string
        environment = "staging"
except:
    environment = "staging"

# Use environment to select catalogs
if environment == "staging":
    catalog = "silver_staging"
    admin_catalog = "dataagg_admin_staging"
else:
    catalog = "silver"
    admin_catalog = "dataagg_admin"

print(f"🔧 Environment: {environment}")
print(f"📊 Using catalog: {catalog}")
```

---

## Staging Data Setup: Shallow Clone Pattern

When notebooks depend on shared tables that already exist in production (admin tables, processing backlogs, lookup tables), create **shallow clones** in staging. This gives staging a realistic snapshot of prod data without copying or modifying the source.

### Why Shallow Clones

Shallow clones solve a common problem: notebooks in staging fail because they expect tables that only exist in production. Rather than manually creating test data or sharing prod tables (risky), shallow clones provide:

- **Instant setup**: No data copy, just metadata
- **Prod-like data**: Staging tests against realistic records
- **Isolation**: Staging writes don't affect prod
- **Fresh snapshots**: Each pipeline run can start from current prod state

### When to Use

Create shallow clones for tables that:

- Must exist before the pipeline can run (admin tables, backlogs, config tables)
- Contain reference data the pipeline reads but doesn't own
- Would cause `TABLE_OR_VIEW_NOT_FOUND` errors in staging if missing

**Don't** clone tables the pipeline creates or writes to — those will be created naturally via `saveAsTable()`.

### Implementation Pattern

Place this code in the **first task** that runs in staging, before any downstream tasks attempt to read the tables:

```python
# Cell 1: Setup staging data dependencies
try:
    dbutils.widgets.text("environment", "staging", "Environment")
    environment = dbutils.widgets.get("environment")
    if not environment:
        environment = "staging"
except:
    environment = "staging"

# Only run cloning in staging
if environment == "staging":
    print("🔧 Setting up staging data...")
    
    # Define catalogs
    prod_admin_catalog = "dataagg_admin"
    staging_admin_catalog = "dataagg_admin_staging"
    
    # Define table mappings (add more as needed)
    tables_to_clone = [
        {
            "prod": f"{prod_admin_catalog}.processing_backlogs.dataagg_dag_backlog",
            "staging": f"{staging_admin_catalog}.processing_backlogs.dataagg_dag_backlog",
            "schema": f"{staging_admin_catalog}.processing_backlogs"
        },
        {
            "prod": f"{prod_admin_catalog}.config.lookup_tables",
            "staging": f"{staging_admin_catalog}.config.lookup_tables",
            "schema": f"{staging_admin_catalog}.config"
        }
    ]
    
    # Clone each table
    for table_config in tables_to_clone:
        prod_table = table_config["prod"]
        staging_table = table_config["staging"]
        schema = table_config["schema"]
        
        try:
            # Ensure schema exists
            spark.sql(f"CREATE SCHEMA IF NOT EXISTS {schema}")
            
            # Drop existing staging table to get fresh snapshot
            spark.sql(f"DROP TABLE IF EXISTS {staging_table}")
            
            # Create shallow clone
            spark.sql(f"CREATE TABLE IF NOT EXISTS {staging_table} SHALLOW CLONE {prod_table}")
            
            # Verify clone succeeded
            count = spark.table(staging_table).count()
            print(f"✅ Cloned {prod_table} → {staging_table} ({count:,} rows)")
            
        except Exception as e:
            print(f"⚠️  Failed to clone {prod_table}: {str(e)}")
            # Decide whether to fail or continue based on criticality
            # raise  # Uncomment to fail the job if this table is critical
    
    print("✅ Staging data setup complete")
else:
    print(f"ℹ️  Production mode - using live tables")
```

### Key Implementation Notes

**Drop before cloning**: Always `DROP TABLE IF EXISTS` before cloning. This ensures staging starts from a fresh prod snapshot rather than stale data from previous runs.

**Run early**: Place this in the first task that executes. If a downstream task tries to read a table before it's cloned, the job fails.

**Environment check**: Only clone in staging (`if environment == "staging"`). Production should use the actual tables directly.

**Error handling**: Decide whether cloning failures should stop the job or just log warnings. Critical dependencies should `raise`, optional ones can continue.

**Schema creation**: Create schemas before tables (`CREATE SCHEMA IF NOT EXISTS`) since staging catalogs may not have the same structure as prod initially.

### Integration with Job Definition

In your job.yml, ensure the cloning task runs first:

```yaml
tasks:
  - task_key: setup_staging_data
    existing_cluster_id: "${var.cluster_id}"
    notebook_task:
      notebook_path: ./setup_staging_data.ipynb
      source: WORKSPACE
      base_parameters:
        environment: "${bundle.target}"
  
  - task_key: main_processing
    depends_on:
      - task_key: setup_staging_data
    existing_cluster_id: "${var.cluster_id}"
    notebook_task:
      notebook_path: ./main_processing.ipynb
      source: WORKSPACE
      base_parameters:
        environment: "${bundle.target}"
```

The `depends_on` ensures data is cloned before processing begins.
