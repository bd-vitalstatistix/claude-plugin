---
name: superduck-dags
description: Create and modify SuperDuck Airflow DAGs for data lake ingestion. Use when users want to add new data sources to the data lake, create DAG config files, modify SQL queries, choose merge strategies, fix merge_on formatting issues, troubleshoot DAG generation, understand write options (scd/overwrite/append/preserve), configure column classifications, adapt existing DAGs to new sources, or set up incremental data pipelines. Also use when encountering composite key formatting errors or when users mention SuperDuck, data lake push, Unity Catalog ingestion, or Airflow DAG generation.
---

# SuperDuck DAG Adaptation

Create and modify Airflow DAGs that push data from source databases to Azure Blob Storage and Unity Catalog using the SuperDuck pipeline framework.

## What This Skill Covers

SuperDuck DAGs handle:
- Pulling data from source databases using SQL queries
- Uploading data to Azure Blob Storage in Parquet format
- Registering datasets in Unity Catalog processing backlog
- Managing schema tracking and metadata
- Configuring merge strategies (SCD, overwrite, append, preserve)
- Incremental and full refresh patterns

**This skill does NOT cover:**
- Airflow infrastructure setup
- Azure storage account configuration
- Unity Catalog administration
- Databricks cluster management

---

## Quick Navigation

**What do you need to do?**

- **Create a new DAG from scratch** → Go to [Creating a New DAG](#creating-a-new-dag)
- **Modify an existing DAG** → Go to [Modifying Existing DAGs](#modifying-existing-dags)
- **Fix merge_on formatting error** → Go to [Critical: merge_on Formatting](#critical-merge_on-formatting)
- **Choose the right write option** → Go to [Write Option Decision Guide](#write-option-decision-guide)
- **Route a dataset to the hourly vs daily backlog job** → Go to [Job Routing (target_job)](#job-routing-target_job)
- **Handle large historical backlogs** → Go to [Handling Large Backlogs](#handling-large-backlogs)
- **View reference documentation** → See [Additional Reference Files](#additional-reference-files)

---

## Critical: merge_on Formatting

**This is the most common configuration error.** The `merge_on` parameter must be formatted differently for single vs composite keys, and the format depends on whether you're using the generator or hand-editing.

### The Problem

The DAG generator (`dag_generator.py`) has special handling for `merge_on` that converts values to the format expected by downstream processing. Getting this wrong causes merge operations to fail silently or use incorrect columns.

### The Solution

**For single-column merge keys:**
```json
"merge_on": "id"
```
Simple string, no brackets.

**For composite (multi-column) merge keys:**
```json
"merge_on": "[\"clientid\", \"daystr\"]"
```
This is a JSON string containing a JSON array. Use escaped quotes.

Or more readably in Python strings:
```python
merge_on = '["clientid", "daystr"]'
```

### What NOT To Do

```json
❌ WRONG - comma-separated string
"merge_on": "clientid,daystr"

❌ WRONG - Python list in JSON (gets stringified incorrectly)
"merge_on": ["clientid", "daystr"]
```

### Real Examples

✅ **Single key**:
```json
"merge_on": "id"
```

✅ **Composite key**:
```json
"merge_on": "[\"clientid\", \"daystr\"]"
```

---

## Write Option Decision Guide

The `write_option` parameter controls how data is written to Delta tables in Unity Catalog. 

### ⚠️ VALID OPTIONS ONLY

**ONLY these 5 values are supported:**
- `"scd"` - Slowly Changing Dimensions Type 2
- `"overwrite"` - Update matching records
- `"preserve"` - Update matches, keep unmatched 
- `"full_overwrite"` - Replace entire table
- `"append"` - Add new records only

**Any other value (including "merge", "upsert", "replace", etc.) is INVALID and will cause errors.**

Choose based on your requirements:

### Decision Tree

```
Do you need to track historical changes over time?
├─ YES → Use "scd" (Slowly Changing Dimensions Type 2)
│         - Keeps full history with is_current flag
│         - merge_on REQUIRED
│
└─ NO → Do you need to update existing records?
    ├─ YES → Do you want to keep unmatched old records?
    │   ├─ YES → Use "preserve"
    │   │         - Updates matches, marks unmatched as not current
    │   │         - merge_on REQUIRED
    │   │
    │   └─ NO → Use "overwrite"
    │             - Deletes matches, inserts new records
    │             - merge_on REQUIRED
    │
    └─ NO → Do you want to replace the entire table?
        ├─ YES → Use "full_overwrite"
        │         - Deletes entire table, loads fresh data
        │         - merge_on NOT USED
        │
        └─ NO → Use "append"
                  - Adds all new records, no updates
                  - merge_on NOT USED
```

### Quick Reference

| Option | History Tracking | Updates Records | Requires merge_on | Best For |
|--------|------------------|-----------------|-------------------|----------|
| **scd** | ✅ Yes | ✅ Yes | ✅ Required | Dimensional data with full history |
| **overwrite** | ❌ No | ✅ Yes | ✅ Required | Incremental updates without history |
| **preserve** | ⚠️ Partial | ✅ Yes | ✅ Required | Hybrid: update + archive unmatched |
| **full_overwrite** | ❌ No | N/A | ❌ Not used | Weekly/monthly full refresh |
| **append** | ❌ No | ❌ No | ❌ Not used | Raw logs, immutable events |

**See `references/write-options.md` for detailed explanations and before/after examples.**

---

## Creating a New DAG

Follow this workflow to add a new data source to the data lake.

### Step 1: Gather Requirements

Before creating config files, collect this information:

**Dataset Identity** (must be unique across Unity Catalog):
- `dataset_name` - Table name (e.g., "instance_stats")
- `dataset_product` - Product/category (e.g., "sca_phonehome_kb")
- Together they form: `{product}.{dataset_name}`

**Source Connection**:
- `data_source` - Database name (e.g., "kb_analytics_blackduck")
- `conn_id` - Airflow connection ID (e.g., "kb_analytics_blackduck_ro")

**Write Strategy**:
- `write_option` - Use decision tree above to choose
- `merge_on` - Column(s) for merge (if using scd/overwrite/preserve)

**Schedule**:
- How often? (hourly=60, daily=1440, weekly=10080 minutes)
- Backfill needed? (set `catchup` true/false)
- Start date (when should this DAG begin?)
- ⚠️ **Large backlog?** If start date is >6 months ago, see [Handling Large Backlogs](#handling-large-backlogs)
- Which backlog-processing job should pick this up? (`target_job`: `"daily"` default, or `"hourly"` — see [Job Routing](#job-routing-target_job))

**SQL Query**:
- What data to pull?
- Date windowing for incremental? (use `{dag_start_date}`, `{dag_end_date}`)

**Optional**:
- Column classifications (public/internal/confidential/restricted)
- Column descriptions (documentation)
- Special notes

### Step 2: Write the SQL Query

Your query should:
- Select only the columns you need
- Use date windowing for incremental loads
- Be compatible with the source database

**Date windowing pattern for incremental loads:**
```sql
SELECT col1, col2, col3, date_col
FROM schema.table
WHERE date_col >= '{dag_start_date}'
  AND date_col < '{dag_end_date}'
```

The placeholders `{dag_start_date}` and `{dag_end_date}` are automatically replaced by Airflow with the DAG's execution date and next execution date.

**For full refresh (no windowing):**
```sql
SELECT col1, col2, col3
FROM schema.table
WHERE active = true
```

**Test your query** against the source database before adding it to config:
```bash
# Example
psql -h source-db.example.com -U readonly -d analytics -c "SELECT ..."
```

### Step 3: Create or Update Config File

Config files live in `config/<product_name>.json`.

**If the product already exists**, add a new entry to the `dags` array.

**If this is a new product**, create a new config file:

```json
{
    "defaults": {
        "schedule": 1440,
        "storage_account": "dataaggbronze",
        "storage_container": "landing-zone",
        "schema_repo": "git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git",
        "dataset_category": "product",
        "datalake_class": "datum",
        "start_year": 2024,
        "start_month": 1,
        "start_day": 1,
        "catchup": true,
        "max_active_runs": 3,
        "write_option": "overwrite",
        "merge_on": "id",
        "column_classification": {},
        "column_description": {},
        "folder": "mlgpu"
    },
    "dags": [
        {
            "dataset_name": "your_dataset_name",
            "dataset_product": "your_product_name",
            "data_source": "your_source_db",
            "conn_id": "your_airflow_connection_id",
            "notes": "Description of what this dataset contains",
            "dag_name": "your_product_your_dataset",
            "query": "SELECT * FROM table WHERE date >= '{dag_start_date}' AND date < '{dag_end_date}'"
        }
    ]
}
```

**Key configuration tips:**

1. **Defaults vs per-DAG overrides**: Set common values in `defaults`, override in specific DAG entries
2. **merge_on format**: Use the correct format from the section above!
3. **folder**: Usually "mlgpu" or "dc2" depending on deployment target
4. **dag_name**: Convention is `{product}_{dataset}` in snake_case

### Step 4: Generate the DAG

Run the generator script:

```bash
cd utilities
python dag_generator.py
```

This will:
- Read all JSON files from `config/`
- Generate DAG files in `../{folder}/dags/generated/{product}/`
- Add generated files to git staging area

**Verify the output:**
```bash
# Check what was generated
git status

# Review the generated DAG
cat ../mlgpu/dags/generated/your_product/your_product_your_dataset.py

# Verify merge_on formatting is correct
grep "merge_on" ../mlgpu/dags/generated/your_product/your_product_your_dataset.py
```

### Step 5: Test and Deploy

**Local testing** (if you have Airflow running locally):
```bash
# Validate DAG syntax
python ../mlgpu/dags/generated/your_product/your_product_your_dataset.py

# List DAG tasks
airflow dags list-tasks your_product_your_dataset
```

**Deploy to Airflow:**
1. Commit the generated DAG file
2. Push to the appropriate branch
3. CI/CD will sync to Airflow

**Test run in Airflow UI:**
1. Navigate to your DAG: `http://your-airflow:8080/dags/your_product_your_dataset/grid`
2. Enable the DAG
3. Trigger a manual run
4. Monitor task logs for errors
5. Verify data landed in Azure: `{storage_container}/{product}/{data_source}/`
6. Check Unity Catalog for processing backlog entry

**Common first-run issues:**
- Connection ID doesn't exist → Create in Airflow UI
- Query syntax error → Test query directly against source DB
- Permissions error → Verify read access to source, write access to Azure
- merge_on error → Check formatting (see section above)
- **Invalid write_option → Must use ONLY: scd, overwrite, preserve, full_overwrite, or append**

### Job Routing (target_job)

Every generated DAG writes a row to the `dataagg_admin.processing_backlogs` table describing the file it just uploaded. That row includes a `target_job` column so downstream Databricks jobs can filter their own queue instead of competing for the same backlog.

**Valid values**: `"daily"` (default) | `"hourly"`

- **`"daily"`** — Picked up by the standard daily backlog-processing job. This is the default if `target_job` is omitted; the generator fills it in automatically via `config.setdefault("target_job", "daily")`, so existing configs need no changes.
- **`"hourly"`** — Picked up by the hourly backlog-processing job instead. Use this for datasets that need faster (sub-daily) turnaround, such as `continuous_dynamic` and `polaris`, which set `"target_job": "hourly"` in their `defaults`.

**Setting it:**
```json
"defaults": {
    "target_job": "hourly"
}
```
Can also be set per-DAG entry to override the config-level default, just like any other parameter.

**Important**: `target_job` only controls which Databricks job *claims* the backlog row — it does not affect the DAG's own Airflow `schedule`. An hourly `schedule` with `target_job: "daily"` (or vice versa) is valid but usually not what you want; keep them aligned unless you have a specific reason not to.

### Handling Large Backlogs

**⚠️ Important for new DAGs with years of historical data:**

If your new DAG has `catchup=true` and a start date far in the past (e.g., 2018), Airflow will attempt to run the DAG once per schedule interval (e.g., daily) for ALL historical dates. For a DAG starting in 2018, this could be **thousands of runs**, which is extremely slow and resource-intensive.

**Solution: Initial Bulk Load**

For new DAGs with large catchup windows, use one of these strategies:

**Option 1: Temporary bulk-load DAG**
1. Create a secondary DAG with the same config BUT:
   - Set `catchup=false`
   - Modify the query to include the FULL date range in one run:
     ```sql
     WHERE daystr >= '2018-12-20' AND daystr < '2024-01-01'
     ```
   - Use a different `dag_name` (e.g., `sca_phonehome_kb_instance_stats_bulk_load`)
2. Run this bulk-load DAG once manually
3. Once complete, delete/disable the bulk-load DAG
4. Deploy the normal incremental DAG with `catchup=true` starting from where bulk load ended

**Option 2: Manual bulk extraction**
1. Run the full-range query manually against the source database
2. Export results to Parquet
3. Upload directly to Azure storage
4. Register in Unity Catalog processing backlog manually
5. Then deploy the incremental DAG

**When to use this:**
- Start date is >6 months in the past
- Catchup would result in >100 individual runs
- You need historical data loaded quickly

**When NOT needed:**
- Recent start date (last few weeks/months)
- `catchup=false` (only runs from today forward)
- Small schedule intervals don't matter (e.g., 10-20 runs to catch up)

---

## Modifying Existing DAGs

To modify an existing DAG:

### 1. Locate the Config File

Generated DAGs have a comment at the top showing which config generated them:
```python
# Generated from config/sca_phonehome_kb.json
```

Or search:
```bash
cd config
grep -l "dag_name.*your_dag_name" *.json
```

### 2. Edit the Config

Make your changes in the JSON file:
- Update query
- Change schedule
- Modify write_option or merge_on
- Add column classifications

### 3. Regenerate

```bash
cd utilities
python dag_generator.py
```

This will overwrite the existing generated DAG with your changes.

### 4. Review and Deploy

```bash
git diff ../mlgpu/dags/generated/your_product/your_dag.py
git add config/your_product.json
git add ../mlgpu/dags/generated/your_product/your_dag.py
git commit -m "Update your_dag: description of changes"
```

### When to Edit Generated Files Directly

**Generally, don't.** Generated files will be overwritten next time the generator runs.

**Exception**: One-off debugging or hotfixes. But document what you changed and why, and update the config file later to match.

---

## Key Parameters Quick Reference

This table covers the most commonly used parameters. For complete details, see `references/parameters.md`.

| Parameter | Required | Type | Description | Example |
|-----------|----------|------|-------------|---------|
| `dataset_name` | Yes | string | Unity table name | `"instance_stats"` |
| `dataset_product` | Yes | string | Product/category | `"sca_phonehome_kb"` |
| `data_source` | Yes | string | Source database | `"kb_analytics_ro"` |
| `conn_id` | Yes | string | Airflow connection ID | `"kb_conn"` |
| `write_option` | Yes | string | Merge strategy | `"scd"` \| `"overwrite"` \| `"append"` \| `"preserve"` \| `"full_overwrite"` |
| `merge_on` | Conditional | string/array | Merge column(s) | `"id"` or `'["col1", "col2"]'` |
| `query` | Yes | string | SQL query | `"SELECT * FROM..."` |
| `schedule` | No | integer | Interval in minutes | `1440` (daily) |
| `start_date` | No | object | DAG start date | `datetime(2024, 1, 1)` |
| `catchup` | No | boolean | Backfill past runs | `true` \| `false` |
| `dag_name` | No | string | DAG identifier | `"product_dataset"` |
| `notes` | No | string | Documentation | `"Daily telemetry..."` |
| `column_classification` | No | object | Column security tags | `{"col": "confidential"}` |
| `column_description` | No | object | Column docs | `{"col": "User ID"}` |
| `target_job` | No | string | Which Databricks backlog-processing job picks up this dataset | `"daily"` (default) \| `"hourly"` |

**⚠️ Important: `write_option` must be EXACTLY one of the 5 values listed above. No other values are valid.**

**Parameter inheritance:**
- Set in `defaults` → applies to all DAGs in that config file
- Set in individual DAG entry → overrides the default
- Not set anywhere → uses hardcoded template default

---

## Common Scenarios

### Scenario 1: Daily Incremental Telemetry Data

**Use case**: Pull yesterday's telemetry data every day, updating existing records

```json
{
    "dataset_name": "instance_stats",
    "dataset_product": "sca_phonehome_kb",
    "data_source": "kb_analytics_blackduck",
    "conn_id": "kb_analytics_blackduck_ro",
    "write_option": "overwrite",
    "merge_on": "[\"clientid\", \"daystr\"]",
    "schedule": 1440,
    "catchup": true,
    "query": "SELECT clientid, daystr, scan_count, project_count FROM stats WHERE daystr >= '{dag_start_date}' AND daystr < '{dag_end_date}'"
}
```

**Why these choices:**
- `write_option: overwrite` - Only need current values, not history
- `merge_on: ["clientid", "daystr"]` - Unique per client per day
- `catchup: true` - Backfill historical data on first run
- Date windowing in query - Incremental loads

### Scenario 2: Multi-Column Merge Key

**Use case**: Scan response data unique by location + slot + instance

```json
{
    "dataset_name": "scan_response_aggregator",
    "dataset_product": "continuous_dynamic",
    "write_option": "overwrite",
    "merge_on": "[\"lid\", \"slot_id\", \"instance_id\"]",
    "query": "SELECT lid, slot_id, instance_id, scan_data, timestamp FROM responses WHERE timestamp >= '{dag_start_date}'"
}
```

**Why these choices:**
- `write_option: overwrite` - Daily updates without history
- `merge_on: ["lid", "slot_id", "instance_id"]` - Composite key, three columns
- Note the JSON array string format for composite keys!

---

## Additional Reference Files

- **`references/lessons-learned.md`** - Historical issues and solutions (merge_on formatting, PostgreSQL quoting)
- **`references/write-options.md`** - Detailed write option explanations with before/after examples
- **`references/parameters.md`** - Complete parameter reference and inheritance rules
- **`references/troubleshooting.md`** - Common errors and debugging steps
- **`references/examples.md`** - Real-world DAG examples

---

## Additional Resources

**Repository locations:**
- DAG generator: `utilities/dag_generator.py`
- Config files: `config/`
- Generated DAGs: `mlgpu/dags/generated/` or `dc2/dags/generated/`
- Template: `templates/SuperDuck_push_template.py`

**Related documentation:**
- Unity Catalog: (internal documentation)
- Airflow connections: (internal documentation)
- Azure storage: (internal documentation)

**Getting help:**
- Check `references/troubleshooting.md` first
- Review logs in Airflow UI
- Ask in data engineering Slack channel
- Consult with senior data engineers for complex cases
