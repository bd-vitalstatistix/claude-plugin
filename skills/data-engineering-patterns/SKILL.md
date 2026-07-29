---
description: End-to-end data engineering patterns — pipeline selection, Airflow DAG → data product lifecycle, ETL job configuration, Databricks bundle structure
disable-model-invocation: false
---

# Data Engineering Patterns

## Pipeline Pattern Selection

Choose the right architecture for a new data source:

```
New data source?
├─ Cloud storage (S3, ADLS), streaming (Kafka)
│  → Full Medallion (3-layer Bronze/Silver/Gold)
├─ External API, legacy DB, file export
│  → Simplified (2-phase Volume → Delta → Views)
│     └─ Need queryable raw data? → Hybrid (add Bronze over Volume)
```

### Simplified Pattern (current standard for API/DB-pull)

```
External Source → Volume (Parquet) → Delta Table (transform) → Views (aggregation)
```

Best for: single team, straightforward transformations, external extraction required.

### Full Medallion (for complex use cases)

```
Source → Auto Loader → Bronze (raw) → Silver (cleaned) → Gold (aggregated)
```

Best for: multiple consumers, multi-stage transforms, variable data quality, regulatory audit trail.

### Migration Path (Simplified → Hybrid)

Add a Bronze table reading from Volume (zero-copy) when you later need queryable raw data:
```sql
CREATE TABLE bronze.my_table USING PARQUET
LOCATION '/Volumes/catalog/schema/source_exports/';
```

## End-to-End Pipeline Lifecycle

### Overview

```
Source DB (Postgres/etc.)
    ↓  Stage 1
Airflow DAG → Azure Blob → Backlog entry
    ↓  Stage 2
`process_dag_backlog_prod` → silver.* Delta tables
    ↓  Stage 3
dbt staging models → data_product_staging.*
    ↓  Stage 4
dbt intermediate/marts → data_product_marts.*
    ↓  Stage 5
`update_catalog_descriptions()` → SuperDuck discoverable
```

### Stage 1: Airflow DAG Configuration

**Files live in** `ds-dags-02/` under the appropriate folder:
- `dc2/` for BD internal network sources (`*.bd.internal`, `*.kb-factory.internal`)
- `mlgpu/` for GPU/ML workloads

**Config-driven generation.** Every source gets:
- `config/<product>.json` — SQL query, schedule, merge key, notes
- `templates/SuperDuck_push_template.py` — structural/behavioural changes
- Generated DAG → `dc2/dags/generated/<product>/` or `mlgpu/dags/generated/<product>/`

**CRITICAL — NEVER edit generated DAG files directly.** They're overwritten every time `utilities/dag_generator.py` runs. Always change the config or template, then regenerate:
```bash
cd ds-dags-02/utilities && uv run --with click dag_generator.py
git add config/<product>.json templates/ dc2/dags/generated/ mlgpu/dags/generated/
git commit -m "..."
```

**Always test your query against the source before committing:**
```bash
psql service=<conn_id> -c "<your query> LIMIT 1;"
```

Check for three gotchas that only surface at runtime:
```sql
-- Reserved words used as bare column names (references, user, table, value)
-- Mixed-case columns requiring double-quoting (CWE, CVSS2, base_CWE_confidence)
-- jsonb/json/ARRAY columns requiring ::text cast for PyArrow
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'YourTable'
  AND (data_type IN ('jsonb', 'json', 'ARRAY')
       OR column_name != lower(column_name));
```

**Large table date-windowing.** The `DBtoAzureParquetOperator` loads the full result set into a pandas DataFrame — tables over ~200MB OOM the worker pod. Fix with date-window filters + catchup:
```json
{
    "query": "SELECT ... WHERE created_at >= '{dag_start_date}' AND created_at < '{dag_end_date}'",
    "catchup": true,
    "max_active_runs": 3,
    "write_option": "merge",
    "merge_on": "id"
}
```

Use `"write_option": "scd"` for append-only audit/history tables, `"merge"` for tables where rows are updated in place.

**Source data quality: corrupt values.** Apply imputation at the DAG query level (not in dbt):
```sql
CASE WHEN status LIKE '<%' THEN 'ASSIGNED' ELSE status END AS status,
CASE WHEN status LIKE '<%' THEN TRUE ELSE FALSE END AS status_was_imputed
```

**New DAGs appear paused** in Airflow by default — enable manually after first sync.

**Do not use `airflow dags backfill`** on Airflow 2.9.x (deadlock bug with `on_failure` callbacks). Use a trigger loop instead:
```bash
for d in $(seq 0 N); do
  airflow dags trigger <dag_id> --exec-date $(date -d "-$d days" +%Y-%m-%dT00:00:00+00:00)
  sleep 5
done
```

### Stage 2: Backlog Processing

**Job**: `process_dag_backlog_prod` on the `dataaggdbx` Databricks workspace  
**Schedule**: every 12 hours (can be manually triggered)

Task pipeline:
1. `discover_backlog` — finds unprocessed backlog entries
2. `process_backlog` — reads Parquet from blob, writes to `silver.<product>.<table>`
3. `syncing_schemas_to_repo` — updates schema tracking
4. `update_column_classifications` — sensitivity labels
5. `update_column_description` — column descriptions

Check pending backlog:
```bash
databricks --profile dataaggdbx api post /api/2.0/sql/statements \
  --json '{
    "warehouse_id": "<warehouse_id>",
    "statement": "SELECT dataset_product, backlog_status, COUNT(*) as n FROM dataagg_admin.processing_backlogs.dataagg_DAG_backlog WHERE processing_job_id IS NULL GROUP BY 1,2",
    "wait_timeout": "50s"
  }'
```

**Diagnostic signal for OOM**: a missing backlog entry on a failed DAG run means the failure happened in `upload_tables` before `add_to_queue` ran.

### Stage 3: dbt Source Declaration

In `uc_transforms/models/staging/<product>/_<product>__sources.yml`:
```yaml
sources:
  - name: trims
    database: silver
    schema: trims
    tables:
      - name: vulnerability
        description: Current BDSA vulnerability snapshot
```

**Key**: `database: silver` points at the Unity Catalog where the backlog processor wrote the data.

### Stage 4: dbt Staging → Intermediate → Marts

```sql
-- Staging model: stg_<product>__<entity>.sql
SELECT
    id as vulnerability_id,
    bdsa_id,
    current_timestamp() as dbt_updated_at
FROM {{ source('trims', 'vulnerability') }}
```

The `dbt_project.yml` catalog routing sends staging models to `data_product_staging.*` in production automatically.

Add `_<product>__models.yml` with column descriptions following the data-product-description-standards skill.

**Cross-reference:** See `/data-science:data-product-description-standards` for standards on grain, relationships, caveats, and how descriptions propagate to SuperDuck via `persist_docs` + `update_catalog_descriptions()`.

### Stage 5: Data Product Registration

Registration into SuperDuck MCP is automatic once:
1. dbt models have run successfully in production
2. `update_catalog_descriptions()` post-hook applied descriptions
3. SuperDuck MCP server redeployed (picks up new catalog)

The data product is queryable via `query_data_product` and discoverable via `list_data_products` / `describe_data_product` at that point.

## ETL Job Configuration (Databricks)

### Cluster Configuration

**Standard (recommended for most ETL)**:
```yaml
job_clusters:
  - job_cluster_key: etl_cluster
    new_cluster:
      spark_version: "14.3.x-scala2.12"
      node_type_id: "Standard_DS3_v2"
      num_workers: 1
```

**Single-node Photon** (for SQL-heavy light ETL):
```yaml
    new_cluster:
      spark_version: "15.4.x-scala2.12"
      node_type_id: "Standard_D4s_v3"
      num_workers: 0
      runtime_engine: "PHOTON"
      spark_conf:
        spark.databricks.cluster.profile: "singleNode"
        spark.master: "local[*]"
        spark.databricks.delta.autoOptimize.optimizeWrite: "true"
        spark.databricks.delta.autoOptimize.autoCompact: "true"
```

**Performance note**: standard multi-worker clusters often outperform single-node Photon for Python ETL workloads despite the faster per-core execution — test both before committing.

### Bundle Structure

```
bundle/
├── databricks.yml              # Targets, variables, resources
├── resources/
│   ├── job.yml                 # Job definition with notebooks
│   └── dashboard.yml           # Lakeview dashboards
└── src/
    └── notebooks/
        ├── 01_setup_schema.py
        ├── 02_ingest_transform.py
        └── 03_validate_summarize.py
```

### Notebook Parameter Alignment

Always match notebook widget defaults to bundle target variables:
```python
dbutils.widgets.text("schema", "sca_phonehome_dev", "Schema")
```
```yaml
targets:
  dev:
    variables:
      schema: sca_phonehome_dev
```

### Cross-Schema Access

Avoid complex cross-schema joins in ETL — use placeholder values instead:
```python
0.0 as customer_acv,  -- populated later by a separate process
```

### Environment Targets

```yaml
targets:
  dev:
    variables:
      schema: sca_phonehome_dev
      source_path: "/Volumes/.../sca_phonehome_dev/source_exports"
  prod:
    variables:
      schema: sca_phonehome
      source_path: "/Volumes/.../sca_phonehome/exports"
```

### Deployment

```bash
databricks bundle validate    # Always validate first
databricks bundle deploy      # Deploy to target
databricks bundle run job_name # Test run
```

## Error Prevention

| Issue | Fix |
|-------|-----|
| Setup schema failures | Keep view creation separate from table creation; test complex views independently |
| Parameter mismatches | Align widget defaults with bundle target variables; test with debug prints |
| Init script failures | Avoid custom init scripts; prefer pre-built cluster images |
| OOM on large tables | Add date-window filters + catchup (Stage 1 above) |
| Jsonb PyArrow type errors | Cast to `::text` in DAG query |
| Missing backlog entry on failure | OOM in `upload_tables` — add date windowing |

## Monitoring

- Track `setup_duration` vs `execution_duration` ratios
- Monitor cluster startup times across runs
- Set up alerts for job failures
- Store entire bundle in git with descriptive commit messages

## Confluence References

| Topic | Link |
|-------|------|
| DAG template parameters | [Quick Start Guide: DAG Template](https://blackduck.atlassian.net/wiki/spaces/DS/pages/2105245716) |
| dbt layer design | [DBT Data Transformation Architecture](https://blackduck.atlassian.net/wiki/spaces/DS/pages/2684289258) |
| Full API/telemetry path | [End-to-End Data Product Creation](https://blackduck.atlassian.net/wiki/spaces/DS/pages/2714665386) |
| CI/CD deploy | [Databricks Pipelines CI/CD Guide](https://blackduck.atlassian.net/wiki/spaces/DS/pages/2942697751) |
| Medallion architecture deep-dive | [medallion-architecture-research.md](https://github.com/bd-vitalstatistix/data_sources/blob/main/guides/medallion-architecture-research.md) |
