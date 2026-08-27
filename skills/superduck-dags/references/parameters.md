# SuperDuck DAG Parameters Reference

Complete reference for all parameters available in SuperDuck DAG configuration files.

## Parameter Categories

1. [Dataset Identification](#dataset-identification)
2. [Connection Configuration](#connection-configuration)
3. [Write Behavior](#write-behavior)
4. [Scheduling](#scheduling)
5. [Column Metadata](#column-metadata)
6. [Query Definition](#query-definition)
7. [Advanced Options](#advanced-options)

---

## Dataset Identification

These parameters define what the dataset is and where it lives in Unity Catalog.

### dataset_name

**Type**: `string`  
**Required**: Yes  
**Set in**: DAG entry (must be unique per DAG)

Unique identifier for the dataset within the product. This becomes the table name in Unity Catalog.

**Requirements:**
- Must be unique when combined with `dataset_product`
- Use snake_case
- No spaces or special characters except underscore
- Should be descriptive but concise

**Examples:**
```json
"dataset_name": "instance_stats"
"dataset_name": "scan_response_aggregator"
"dataset_name": "capec_version"
```

**Unity Catalog path**: `{catalog}.{dataset_product}.{dataset_name}`

### dataset_product

**Type**: `string`  
**Required**: Yes  
**Set in**: DAG entry or defaults

Product or category this dataset belongs to. This becomes the schema name in Unity Catalog.

**Requirements:**
- Must be unique when combined with `dataset_name`
- Use snake_case
- Groups related datasets together

**Examples:**
```json
"dataset_product": "sca_phonehome_kb"
"dataset_product": "continuous_dynamic"
"dataset_product": "polaris"
```

**Convention**: Often matches the config file name (`config/sca_phonehome_kb.json`)

### data_source

**Type**: `string`  
**Required**: Yes  
**Set in**: DAG entry or defaults

The database or source system from which data is pulled. Used for organization and path naming.

**Examples:**
```json
"data_source": "kb_analytics_blackduck"
"data_source": "sentinel_results_read_only_replica_SC"
"data_source": "gcp-sig-datainsight-shared-prd"
```

**Usage**: Becomes part of the blob storage path: `{container}/{dataset_product}/{data_source}/`

---

## Connection Configuration

### conn_id

**Type**: `string`  
**Required**: Yes  
**Set in**: DAG entry or defaults

Airflow connection ID for the source database. Must exist in Airflow before the DAG runs.

**Examples:**
```json
"conn_id": "kb_analytics_blackduck_ro"
"conn_id": "sentinel_results_SC"
"conn_id": "gcp-sig-datainsight-shared-prd"
```

**How to create connections:**
1. Airflow UI → Admin → Connections
2. Add new connection with this ID
3. Configure host, port, schema, login, password

**Naming convention**: Usually ends with `_ro` for read-only connections

### storage_account

**Type**: `string`  
**Required**: No (has default)  
**Default**: `"dataaggbronze"`  
**Set in**: defaults (rarely overridden)

Azure storage account name where Parquet files are uploaded.

**When to override**: Almost never. The default is the standard data lake landing zone.

### storage_container

**Type**: `string`  
**Required**: No (has default)  
**Default**: `"landing-zone"`  
**Set in**: defaults (rarely overridden)

Azure blob storage container within the storage account.

**When to override**: Rarely. Use default unless you have a specific container for your product.

---

## Write Behavior

### write_option

**Type**: `string` (enum)  
**Required**: Yes  
**Valid values**: `"scd"` | `"overwrite"` | `"full_overwrite"` | `"preserve"` | `"append"`  
**Set in**: DAG entry or defaults

Controls how data is written to the Delta table in Unity Catalog.

**Detailed breakdown:**

#### "scd" - Slowly Changing Dimensions Type 2

**Behavior:**
- Matches records on `merge_on` column(s)
- Marks old records with `is_current = false`
- Inserts new records with `is_current = true`
- Retains full history of all changes

**Requires**: `merge_on`

**Use when:**
- You need to answer "what did this record look like on date X?"
- Regulatory/audit requirements
- Dimensional data (customers, products, accounts)

**Example:**
```json
"write_option": "scd",
"merge_on": "customer_id"
```

#### "overwrite" - Replace Matching Records

**Behavior:**
- Matches records on `merge_on` column(s)
- Deletes matched records
- Inserts new records
- Unmatched records remain unchanged

**Requires**: `merge_on`

**Use when:**
- You only care about current state
- Incremental daily/hourly updates
- History not needed

**Example:**
```json
"write_option": "overwrite",
"merge_on": "[\"clientid\", \"daystr\"]"
```

#### "full_overwrite" - Replace Entire Table

**Behavior:**
- Deletes entire table contents
- Inserts all new records
- Fresh start every run

**Does NOT use**: `merge_on`

**Use when:**
- Weekly/monthly full refresh
- Source provides complete snapshot
- Small tables where full reload is faster than merge

**Example:**
```json
"write_option": "full_overwrite"
```

#### "preserve" - Update and Archive

**Behavior:**
- Matches records on `merge_on` column(s)
- Updates matched records as current
- Marks unmatched records with `is_current = false`
- Keeps unmatched records (soft delete)

**Requires**: `merge_on`

**Use when:**
- You want to track deletions
- Soft-delete pattern
- Need to know when records disappeared

**Example:**
```json
"write_option": "preserve",
"merge_on": "user_id"
```

#### "append" - Add Only, Never Update

**Behavior:**
- Inserts all new records
- Never updates or deletes existing records
- No deduplication

**Does NOT use**: `merge_on`

**Use when:**
- Immutable event logs
- Append-only data (clickstream, application logs)
- Each run produces new records, never updates

**Example:**
```json
"write_option": "append"
```

### merge_on

**Type**: `string` (single column) or `string` (JSON array for composite)  
**Required**: Conditional (required for scd, overwrite, preserve)  
**Set in**: DAG entry or defaults

Column(s) used to match records when merging data.

**Format rules:**

**Single column:**
```json
"merge_on": "id"
```

**Multiple columns (composite key):**
```json
"merge_on": "[\"col1\", \"col2\", \"col3\"]"
```

**IMPORTANT**: For composite keys, use JSON array string format with escaped quotes!

**Common mistakes:**
```json
❌ "merge_on": "col1,col2"              // Wrong: comma-separated
❌ "merge_on": ["col1", "col2"]         // Wrong: raw array
❌ "merge_on": "["col1", "col2"]"       // Wrong: unescaped quotes
✅ "merge_on": "[\"col1\", \"col2\"]"   // Correct: JSON array string
```

**Column requirements:**
- Must exist in both source query and target table
- Should be unique (or unique combination for composite)
- Should not contain nulls
- Should be indexed in target table for performance

**Examples from real DAGs:**
```json
// Single key
"merge_on": "weakness_id"

// Two columns
"merge_on": "[\"slot_id\", \"lid\"]"

// Three columns
"merge_on": "[\"lid\", \"slot_id\", \"instance_id\"]"
```

---

## Scheduling

### schedule

**Type**: `integer` (minutes)  
**Required**: No (has default)  
**Default**: `1440` (daily)  
**Set in**: defaults or DAG entry

How often the DAG runs, in minutes.

**Common values:**
```json
"schedule": 60          // Hourly
"schedule": 1440        // Daily (24 × 60)
"schedule": 10080       // Weekly (7 × 24 × 60)
"schedule": 43200       // Monthly (30 × 24 × 60, approximate)
```

**Also accepts**: Cron expressions or Airflow schedule strings in the template (though config uses integer minutes).

### start_year, start_month, start_day

**Type**: `integer`  
**Required**: No (has default)  
**Defaults**: `2024`, `1`, `1`  
**Set in**: defaults or DAG entry

When the DAG should start running. Combined into `start_date = datetime(start_year, start_month, start_day)`.

**Use cases:**
- Set to the earliest data available in source
- Set to when you want backfill to begin
- Set to current date if no backfill needed

**Example:**
```json
"start_year": 2018,
"start_month": 12,
"start_day": 20
```

### catchup

**Type**: `boolean`  
**Required**: No (has default)  
**Default**: `true`  
**Set in**: defaults or DAG entry

Whether Airflow should backfill DAG runs between `start_date` and now.

**`true`**: Backfill all past runs
- Use when you need historical data
- Each run processes its date window
- Can take a long time for old start dates

**`false`**: Only run going forward from now
- Use for new DAGs where history isn't needed
- Use for event/log data where backfill is irrelevant

**Example:**
```json
"catchup": true   // Backfill from start_date to now
"catchup": false  // Only run from today forward
```

### max_active_runs

**Type**: `integer`  
**Required**: No (has default)  
**Default**: `3`  
**Set in**: defaults or DAG entry

Maximum number of concurrent DAG runs allowed.

**Use cases:**
- Default `3` allows some parallelism during backfill
- Set to `1` if order matters or source DB can't handle parallel queries
- Increase if backfill is slow and source can handle more load

**Example:**
```json
"max_active_runs": 1   // Process one date at a time
"max_active_runs": 5   // Allow more parallelism
```

---

## Column Metadata

### column_classification

**Type**: `object` (dictionary)  
**Required**: No (has default)  
**Default**: `{}`  
**Set in**: defaults or DAG entry

Security classification tags for columns. Used by data governance processes.

**Valid values**: `"public"` | `"internal"` | `"confidential"` | `"restricted"`

**Default if not specified**: All columns tagged as `"internal"`

**Can be:**
- Fully populated (all columns)
- Sparsely populated (only sensitive columns)
- Empty object (all columns use default)
- Omitted entirely

**Example:**
```json
"column_classification": {
    "user_id": "public",
    "email": "confidential",
    "purchase_date": "internal",
    "ssn": "restricted"
}
```

**Sparse example:**
```json
"column_classification": {
    "email": "confidential",
    "ssn": "restricted"
}
// Other columns default to "internal"
```

**Reference**: [Data Classification Policy](https://blackduck.atlassian.net/wiki/spaces/DS/pages/175606367)

### column_description

**Type**: `object` (dictionary)  
**Required**: No (has default)  
**Default**: `{}`  
**Set in**: defaults or DAG entry

Human-readable descriptions of columns. Used for documentation and data catalog.

**Can be:**
- Fully populated (all columns documented)
- Sparsely populated (only important/unclear columns)
- Empty object (no descriptions)

**Example:**
```json
"column_description": {
    "user_id": "Unique identifier for user accounts",
    "email": "User's primary email address",
    "purchase_date": "Timestamp when purchase was completed",
    "amount_usd": "Purchase amount in US dollars"
}
```

---

## Query Definition

### query

**Type**: `string` (SQL)  
**Required**: Yes  
**Set in**: DAG entry (must be unique per DAG)

SQL query that retrieves data from the source database.

**Placeholders available:**
- `{dag_start_date}` - Start of the execution window (YYYY-MM-DD)
- `{dag_end_date}` - End of the execution window (YYYY-MM-DD)

**Best practices:**

1. **Use date windowing for incremental loads:**
```sql
SELECT col1, col2, date_col
FROM schema.table
WHERE date_col >= '{dag_start_date}'
  AND date_col < '{dag_end_date}'
```

2. **Select only needed columns:**
```sql
-- ❌ Don't do this
SELECT * FROM huge_table

-- ✅ Do this
SELECT id, name, value, timestamp FROM huge_table
```

3. **Test before deploying:**
```bash
# Replace placeholders with actual dates
psql -h host -U user -d db -c "SELECT ... WHERE date >= '2024-01-01' AND date < '2024-01-02'"
```

4. **For full refresh, no date filter needed:**
```sql
SELECT col1, col2
FROM schema.table
WHERE active = true
```

5. **Complex queries are fine:**
```sql
SELECT 
    t1.id,
    t1.name,
    t2.category,
    COUNT(t3.events) as event_count
FROM schema.table1 t1
JOIN schema.table2 t2 ON t1.category_id = t2.id
LEFT JOIN schema.table3 t3 ON t1.id = t3.table1_id
    AND t3.event_date >= '{dag_start_date}'
    AND t3.event_date < '{dag_end_date}'
WHERE t1.active = true
GROUP BY t1.id, t1.name, t2.category
```

**Database compatibility:**
- Must be valid SQL for the source database type (PostgreSQL, MySQL, etc.)
- Use database-specific functions if needed
- Avoid vendor-specific syntax if possible for portability

**PostgreSQL identifier quoting (IMPORTANT):**

PostgreSQL requires double quotes for case-sensitive identifiers and reserved words. This is a common source of errors.

**When to quote identifiers in PostgreSQL:**
```sql
-- Mixed-case table names
SELECT * FROM "TRIMSserver_capec"

-- Mixed-case column names  
SELECT "base_CWE_confidence" FROM table

-- SQL reserved words
SELECT id, "references", "user" FROM table

-- Names with special characters
SELECT "column-name" FROM "my table"
```

**In JSON config, escape the quotes:**
```json
"query": "SELECT * FROM \"TRIMSserver_capec\""
"query": "SELECT \"base_CWE_confidence\", \"references\" FROM \"Table\""
```

**Common mistakes:**
```sql
❌ TRIMSserver_capec           -- Folds to lowercase, table not found
❌ `TRIMSserver_capec`          -- Backticks are MySQL, not PostgreSQL
❌ 'TRIMSserver_capec'          -- Single quotes are for strings, not identifiers
✅ "TRIMSserver_capec"          -- Correct PostgreSQL syntax
```

**Testing:** Always test queries in psql first:
```bash
psql -h host -U user -d db
\d                              # List tables (shows actual case)
\d "TRIMSserver_capec"         # Describe table
SELECT * FROM "Table" LIMIT 1; # Test query
```

**See also:** `references/troubleshooting.md` Issue 4.5 for detailed troubleshooting.

---

## Advanced Options

### dag_name

**Type**: `string`  
**Required**: No (auto-derived from file name if not set)  
**Set in**: DAG entry

Unique identifier for the DAG in Airflow. Used in UI, logs, and CLI.

**Convention**: `{dataset_product}_{dataset_name}` in snake_case

**Auto-derived**: If not specified, uses the Python file name (without .py)

**Example:**
```json
"dag_name": "sca_phonehome_kb_instance_stats"
```

**When to override**: Rarely. The auto-derived value is usually correct.

### notes

**Type**: `string`  
**Required**: No  
**Set in**: DAG entry or defaults

Free-text documentation for the DAG. Appears in generated code as a comment.

**Use for:**
- What the dataset contains
- Why certain choices were made
- Special considerations
- Business context

**Example:**
```json
"notes": "Daily SCA instance telemetry from feedback.blackduck_stats_new. One row per (clientid, daystr). clientid is a SHA-256 hash of the registration ID — no reverse-mapping to customer identity available in this DB. Date-windowed by daystr for incremental backfill. merge_on=(clientid,daystr) handles late-arriving corrections."
```

### dataset_category

**Type**: `string`  
**Required**: No (has default)  
**Default**: `"product"`  
**Set in**: defaults (rarely overridden)

High-level categorization of the dataset. Used for organization and file naming.

**Common values**: `"product"`, `"platform"`, `"external"`

**When to override**: Rarely. Most datasets are `"product"`.

### datalake_class

**Type**: `string`  
**Required**: No (has default)  
**Default**: `"datum"`  
**Set in**: defaults (rarely overridden)

Classification for data lake organization. Used by downstream processing.

**When to override**: Only if instructed by data engineering team.

### schema_repo

**Type**: `string`  
**Required**: No (has default)  
**Default**: `"git@github.com:bd-vitalstatistix/superduck-lake-schema-tracking.git"`  
**Set in**: defaults (rarely overridden)

Git repository URL for schema tracking. Used by downstream processes to track schema evolution.

**When to override**: Never, unless the organization changes schema repos.

### folder

**Type**: `string`  
**Required**: No (has default)  
**Default**: `"mlgpu"`  
**Valid values**: `"mlgpu"` | `"dc2"`  
**Set in**: defaults

Which deployment folder the generated DAGs should go into.

**Determines output path**: `../{folder}/dags/generated/{product}/`

**When to override**: Set to `"dc2"` if deploying to the DC2 Airflow instance instead of MLGPU.

### target_job

**Type**: `string` (enum)  
**Required**: No (has default)  
**Default**: `"daily"`  
**Valid values**: `"daily"` | `"hourly"`  
**Set in**: defaults or DAG entry

Routes the DAG's processing-backlog row to the correct Databricks job. When the generated DAG inserts into `dataagg_admin.processing_backlogs.{table}`, it now writes a `target_job` value alongside the other columns so the hourly and daily Databricks backlog-processing jobs can each filter to their own queue instead of racing over the same rows.

**Backward compatibility**: The generator calls `config.setdefault("target_job", "daily")`, so any config file that omits this key is unaffected and continues to be processed by the daily job.

**Example:**
```json
"target_job": "hourly"
```

**Known usage**: `continuous_dynamic` and `polaris` configs set `target_job="hourly"` in their `defaults` because those datasets need faster turnaround. All other products default to `"daily"`.

**When to override**: Only when a dataset needs to be processed by the hourly Databricks job instead of the daily one. Don't set this unless you know the downstream job has been split this way — check with data engineering if unsure.

---

## Parameter Inheritance

Parameters can be set in two places:

1. **`defaults` section**: Applies to all DAGs in the config file
2. **Individual DAG entry**: Overrides the default for that specific DAG

**Example:**
```json
{
    "defaults": {
        "schedule": 1440,
        "write_option": "overwrite",
        "merge_on": "id",
        "catchup": true
    },
    "dags": [
        {
            "dataset_name": "daily_stats",
            // Uses defaults: schedule=1440, write_option=overwrite, catchup=true
            "merge_on": "[\"client_id\", \"date\"]"  // Overrides default
        },
        {
            "dataset_name": "weekly_summary",
            "schedule": 10080,        // Overrides default
            "write_option": "append", // Overrides default
            "catchup": false          // Overrides default
            // merge_on not needed for append
        }
    ]
}
```

**Best practice**: Set common values in defaults, override only what's different per DAG.

---

## Required vs Optional Summary

**Always required:**
- `dataset_name`
- `dataset_product`
- `data_source`
- `conn_id`
- `write_option`
- `query`

**Conditionally required:**
- `merge_on` - Required for scd, overwrite, preserve (not for append, full_overwrite)

**Optional with defaults:**
- `schedule` (default: 1440)
- `start_year`, `start_month`, `start_day` (default: 2024-01-01)
- `catchup` (default: true)
- `max_active_runs` (default: 3)
- `storage_account` (default: "dataaggbronze")
- `storage_container` (default: "landing-zone")
- `dataset_category` (default: "product")
- `datalake_class` (default: "datum")
- `schema_repo` (default: schema tracking repo)
- `folder` (default: "mlgpu")
- `target_job` (default: "daily")

**Optional with no default:**
- `dag_name` (auto-derived from file name)
- `notes`
- `column_classification` (empty object)
- `column_description` (empty object)
