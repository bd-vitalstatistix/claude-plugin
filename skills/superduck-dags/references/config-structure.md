# Config Structure and DAG Generator

Understanding the JSON configuration structure and how the DAG generator works.

---

## Config File Structure

Config files live in `config/<product_name>.json` and define one or more related DAGs.

### Basic Structure

```json
{
    "defaults": {
        // Common parameters inherited by all DAGs
    },
    "dags": [
        {
            // First DAG definition
        },
        {
            // Second DAG definition
        }
    ]
}
```

### The defaults Section

Parameters in `defaults` are inherited by all DAGs in the `dags` array unless overridden.

**Common defaults to set:**
```json
"defaults": {
    "schedule": 1440,                    // All DAGs run daily
    "storage_account": "dataaggbronze",  // Standard storage account
    "storage_container": "landing-zone", // Standard container
    "schema_repo": "git@github.com:...", // Schema tracking repo
    "dataset_category": "product",       // Dataset category
    "datalake_class": "datum",           // Datalake classification
    "start_year": 2024,                  // All DAGs start same date
    "start_month": 1,
    "start_day": 1,
    "catchup": true,                     // Backfill enabled
    "max_active_runs": 3,                // Parallelism limit
    "write_option": "overwrite",         // Default merge strategy
    "merge_on": "id",                    // Default merge key
    "column_classification": {},         // Empty by default
    "column_description": {},            // Empty by default
    "folder": "mlgpu",                   // Target deployment folder
    "target_job": "daily"                // Which backlog job picks this up: "daily" (default) | "hourly"
}
```

**What to put in defaults:**
- Values shared by all DAGs in this config
- Storage/deployment settings (rarely change)
- Common schedule/write_option (if applicable)

**What NOT to put in defaults:**
- DAG-specific values (dataset_name, query, conn_id)
- Values that differ between DAGs

### The dags Array

Each object in the `dags` array defines one DAG. Required fields that are almost always unique:

```json
{
    "dataset_name": "instance_stats",        // Unique table name
    "dataset_product": "sca_phonehome_kb",   // Product schema
    "data_source": "kb_analytics",           // Source database
    "conn_id": "kb_analytics_ro",            // Airflow connection
    "dag_name": "sca_phonehome_kb_instance_stats",  // Unique DAG ID
    "query": "SELECT ... WHERE ...",         // SQL query
    "notes": "What this dataset contains"    // Documentation
}
```

**Optional overrides:**
```json
{
    // ... required fields ...
    "schedule": 60,                          // Override default (hourly)
    "write_option": "scd",                   // Override default
    "merge_on": "[\"col1\", \"col2\"]",      // Override default
    "start_year": 2018,                      // Different start date
    "catchup": false                         // Override default
}
```

### Complete Example

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
            "dataset_name": "daily_stats",
            "dataset_product": "analytics",
            "data_source": "analytics_db_ro",
            "conn_id": "analytics_db_connection",
            "dag_name": "analytics_daily_stats",
            "query": "SELECT user_id, date, action_count FROM stats WHERE date >= '{dag_start_date}' AND date < '{dag_end_date}'",
            "notes": "Daily user activity statistics"
        },
        {
            "dataset_name": "weekly_summary",
            "dataset_product": "analytics",
            "data_source": "analytics_db_ro",
            "conn_id": "analytics_db_connection",
            "dag_name": "analytics_weekly_summary",
            "schedule": 10080,                           // Override: weekly
            "write_option": "full_overwrite",            // Override: full refresh
            "catchup": false,                            // Override: no backfill
            "query": "SELECT category, SUM(value) as total FROM aggregated_data GROUP BY category",
            "notes": "Weekly rollup summary"
        }
    ]
}
```

---

## DAG Generator Workflow

The generator script (`utilities/dag_generator.py`) reads config files and produces DAG Python files.

### How It Works

1. **Load template**: Reads `templates/SuperDuck_push_template.py`
2. **Find configs**: Scans `config/*.json` for all config files
3. **For each config file**:
   - Parse JSON
   - Extract defaults
   - For each DAG in the dags array:
     - Merge defaults with DAG-specific values
     - Replace placeholders in template
     - Write generated DAG to output path
     - Add to git staging

### Template Placeholders

The template uses `{{placeholder}}` syntax that gets replaced:

**Template excerpt:**
```python
dataset_name = "{{dataset_name}}"
dataset_product = "{{dataset_product}}"
data_source = "{{data_source}}"
conn_id = "{{conn_id}}"
write_option = "{{write_option}}"
merge_on = {{merge_on}}
notes = "{{notes}}"
```

**After generation (example):**
```python
dataset_name = "instance_stats"
dataset_product = "sca_phonehome_kb"
data_source = "kb_analytics_blackduck"
conn_id = "kb_analytics_blackduck_ro"
write_option = "overwrite"
merge_on = '["clientid", "daystr"]'
notes = "Daily SCA instance telemetry..."
```

### Special Handling: merge_on

The generator has special logic for `merge_on` (lines 28-36):

```python
if key == "merge_on":
    if isinstance(value, list):
        # Python list → JSON string format
        value_str = str(value).replace("'", '"')
        value_str = f"'{value_str}'"
    else:
        # String → quoted string
        value_str = f"'{value}'"
```

**Input → Output examples:**

```json
// Config has:
"merge_on": "id"
// Generated code:
merge_on = 'id'

// Config has:
"merge_on": "[\"col1\", \"col2\"]"
// Generated code:
merge_on = '["col1", "col2"]'
```

This is why composite keys must use the JSON array string format in config!

### Output Path Structure

Generated DAGs go to: `../{folder}/dags/generated/{product}/`

**Example:**
- Config file: `config/sca_phonehome_kb.json`
- folder: `"dc2"`
- dataset_product: `"sca_phonehome_kb"`
- dataset_name: `"instance_stats"`

**Output**: `../dc2/dags/generated/sca_phonehome_kb/sca_phonehome_kb_instance_stats.py`

### Running the Generator

```bash
cd utilities
python dag_generator.py
```

**Output:**
```
Added to git: ../mlgpu/dags/generated/analytics/analytics_daily_stats.py
Added to git: ../mlgpu/dags/generated/analytics/analytics_weekly_summary.py

Generation complete. Total DAGs generated: 2
All generated files have been added to git staging area.
Use 'git status' to see the changes.
Use 'git commit' to commit the changes when ready.
```

### Generator Script Internals

**Key functions:**

1. `load_template(path)` - Reads template file
2. `load_config(path)` - Parses JSON config
3. `render_dag(template, defaults, dag_config)` - Replaces placeholders
4. `get_config_files(dir)` - Finds all .json files
5. `add_to_git(path)` - Stages generated file
6. `generate_dags(template_path, config_dir)` - Main orchestration

**Error handling:**
- Invalid JSON → Skips file, prints error
- Missing required key → Skips DAG, prints error
- File write errors → Prints error, continues

### Validation

The generator does minimal validation. You should verify:

**After generation:**
```bash
# Check syntax
python ../mlgpu/dags/generated/product/dag_name.py

# Or use Airflow CLI
airflow dags list-import-errors
```

**Common issues:**
- Invalid Python in generated code (usually from malformed config)
- Missing required parameters
- SQL syntax errors (won't be caught until runtime)

---

## Multi-DAG Products

When to group multiple DAGs in one config file?

### Group When:

- **Related datasets from same source**
  - Example: `trims.json` defines DAGs for capec, cwe, exploit_db, etc.
  - All from TRIMS database
  - Share common defaults (connection, schedule)

- **Different tables from same product**
  - Example: `sca_phonehome_kb.json` could have instance_stats, version_stats, etc.
  - All part of SCA phone-home product
  - Natural grouping

- **Same deployment/maintenance cycle**
  - Updated together
  - Reviewed together
  - Similar SLAs

### Separate When:

- **Different sources/connections**
  - Each source might have different config
  
- **Different teams/owners**
  - Easier to manage permissions

- **Different schedules/patterns**
  - Though you can override in DAG entry

- **Independent lifecycles**
  - One product stable, another experimental

### Example: TRIMS (Good Multi-DAG Config)

```json
{
    "defaults": {
        "schedule": 10080,              // Weekly for all
        "data_source": "trims",         // Same source
        "conn_id": "trims_readonly",    // Same connection
        "write_option": "full_overwrite", // All full refresh
        "catchup": false,               // No backfill
        "folder": "dc2"
    },
    "dags": [
        {
            "dataset_name": "capec",
            "dataset_product": "trims",
            "dag_name": "trims_capec",
            "query": "SELECT * FROM capec.current_version"
        },
        {
            "dataset_name": "cwe",
            "dataset_product": "trims",
            "dag_name": "trims_cwe",
            "query": "SELECT * FROM cwe.current_version"
        },
        {
            "dataset_name": "exploit_db",
            "dataset_product": "trims",
            "dag_name": "trims_exploit_db",
            "query": "SELECT * FROM exploit_db.exploits"
        }
    ]
}
```

All TRIMS DAGs share schedule, connection, write strategy. Only differ in table/query.

---

## Folder Structure

### Repository Layout

```
ds-dags-02/
├── config/                         # Config files
│   ├── sca_phonehome_kb.json
│   ├── trims.json
│   ├── polaris.json
│   └── continuous_dynamic.json
│
├── utilities/
│   └── dag_generator.py            # Generator script
│
├── templates/
│   └── SuperDuck_push_template.py  # DAG template
│
├── mlgpu/                          # Production Airflow (MLGPU)
│   └── dags/
│       └── generated/
│           ├── sca_phonehome_kb/
│           │   └── sca_phonehome_kb_instance_stats.py
│           ├── polaris/
│           │   ├── polaris_issue_category_KSA.py
│           │   └── polaris_issue_category_PRD.py
│           └── continuous_dynamic/
│               └── cd_scan_response_aggregator_US.py
│
└── dc2/                            # Secondary Airflow (DC2)
    └── dags/
        └── generated/
            └── trims/
                ├── trims_capec.py
                ├── trims_cwe.py
                └── trims_exploit_db.py
```

### Generated DAG Organization

Each product gets its own subdirectory:
- `generated/{product}/` - Contains all DAGs for that product
- One Python file per DAG
- File name matches `dag_name` parameter

### Choosing mlgpu vs dc2

Set in config:
```json
"defaults": {
    "folder": "mlgpu"  // or "dc2"
}
```

**mlgpu**: Primary production Airflow instance
**dc2**: Secondary Airflow instance (less common)

Most DAGs use `mlgpu`.

---

## Advanced: Custom Templates

The template (`templates/SuperDuck_push_template.py`) can be customized, but be careful:

### Safe Modifications

- **Add new placeholders** (and update configs to provide values)
- **Adjust imports**
- **Add helper functions**
- **Modify task retry logic**

### Risky Modifications

- **Remove core tasks** (breaks pipeline)
- **Change parameter names** (breaks existing configs)
- **Alter placeholder syntax** (breaks generator)

### Adding a New Placeholder

1. **Edit template:**
```python
# Add new parameter
custom_param = "{{custom_param}}"
```

2. **Update config defaults:**
```json
"defaults": {
    "custom_param": "default_value"
}
```

3. **Regenerate all DAGs:**
```bash
cd utilities
python dag_generator.py
```

All DAGs will now include the new parameter.

---

## Troubleshooting Generation

### Config File Not Found

**Symptom:**
```
No JSON configuration files found in config/
```

**Cause**: Running generator from wrong directory

**Fix:**
```bash
cd utilities
python dag_generator.py
```

### Invalid JSON

**Symptom:**
```
Error: Invalid JSON in config/product.json
```

**Cause**: Syntax error in JSON (missing comma, bracket, quote)

**Fix**: Validate JSON:
```bash
python -m json.tool config/product.json
```

Or use an online JSON validator.

### Missing Required Key

**Symptom:**
```
Error in DAG configuration for dag_name: Missing key 'dataset_name'
```

**Cause**: Required parameter not in defaults or DAG entry

**Fix**: Add to DAG entry or defaults:
```json
"dataset_name": "my_dataset"
```

### Placeholder Not Replaced

**Symptom**: Generated file contains `{{placeholder}}`

**Cause**: Parameter not in config (neither defaults nor DAG entry)

**Fix**: Add missing parameter to config

### Generated File Has Syntax Error

**Symptom**: Python syntax error in generated file

**Cause**: Usually invalid characters in config values (unescaped quotes, newlines in strings)

**Fix**: Escape special characters or use proper JSON encoding:
```json
"notes": "Description with \"quotes\""
"query": "SELECT col1, col2 FROM table"
```

### merge_on Format Wrong

**Symptom**: DAG runs but merge fails silently

**Cause**: Composite key formatted as comma-separated string

**Fix**: Use JSON array string format:
```json
"merge_on": "[\"col1\", \"col2\"]"
```

See main SKILL.md for detailed merge_on formatting guide.

---

## Best Practices

### Config File Organization

1. **One config per product** (usually)
2. **Meaningful file names** (`product_name.json`)
3. **Comment-friendly** (JSON doesn't allow comments, but descriptive keys help)
4. **Use defaults liberally** (DRY principle)
5. **Override sparingly** (only when needed)

### Parameter Naming

- **dataset_name**: snake_case, descriptive
- **dag_name**: `{product}_{dataset}` convention
- **conn_id**: matches Airflow connection exactly
- **data_source**: consistent with source system

### Version Control

- **Commit config changes** before generating
- **Review generated diffs** before committing
- **Use meaningful commit messages**: "Add trims_cwe DAG for CVE tracking"
- **Group related changes**: Config + generated files in same commit

### Testing

1. **Validate config JSON** before generating
2. **Generate and review diffs**
3. **Test in dev/staging** before production
4. **Trigger manual DAG run** to validate
5. **Monitor first few runs** for errors

### Documentation

- **Use notes field** for context
- **Document unusual choices** in commit messages
- **Keep README updated** if adding new products
- **Share knowledge** with team when patterns change
