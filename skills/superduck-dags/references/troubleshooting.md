# Troubleshooting Guide

Common issues when working with SuperDuck DAGs and how to resolve them.

---

## Issue 1: merge_on Formatting Error

### Symptoms

- DAG generates successfully
- DAG runs without Python errors  
- Merge operation fails silently or uses wrong columns
- Data duplicates instead of merging
- Unity Catalog shows unexpected merge behavior

### Cause

Composite merge keys formatted as comma-separated string instead of JSON array string.

### Diagnosis

Check the generated DAG file:

```python
# ❌ WRONG - This indicates the problem
merge_on = 'clientid,daystr'

# ✅ CORRECT - Should look like this
merge_on = '["clientid", "daystr"]'
```

### Fix

1. Open config file: `config/<product_name>.json`
2. Find the `merge_on` parameter
3. Fix the format:

```json
// Before (wrong):
"merge_on": "clientid,daystr"

// After (correct):
"merge_on": "[\"clientid\", \"daystr\"]"
```

4. Regenerate:
```bash
cd utilities
python dag_generator.py
```

5. Verify generated file has correct format
6. Redeploy and rerun DAG

### Prevention

- Always use JSON array string format for composite keys
- Single keys: `"merge_on": "id"`
- Multiple keys: `"merge_on": "[\"col1\", \"col2\"]"`
- Review generated file after generation

### Reference

See main SKILL.md section "Critical: merge_on Formatting" for detailed explanation.

---

## Issue 2: DAG Generation Fails

### Symptom: "No JSON configuration files found"

**Output:**
```
No JSON configuration files found in config/
```

**Cause**: Running generator from wrong directory

**Fix:**
```bash
# Check current directory
pwd

# Should be in utilities/
cd utilities
python dag_generator.py
```

### Symptom: "Invalid JSON"

**Output:**
```
Error: Invalid JSON in config/product.json
```

**Cause**: JSON syntax error (missing comma, bracket, unmatched quote)

**Fix:**

Validate JSON syntax:
```bash
# Use Python JSON tool
python -m json.tool config/product.json

# Or use jq if installed
jq . config/product.json
```

Common JSON errors:
```json
// ❌ Missing comma
{
    "key1": "value1"
    "key2": "value2"
}

// ✅ Fixed
{
    "key1": "value1",
    "key2": "value2"
}

// ❌ Trailing comma
{
    "key1": "value1",
    "key2": "value2",
}

// ✅ Fixed (no trailing comma)
{
    "key1": "value1",
    "key2": "value2"
}

// ❌ Unescaped quotes in string
{
    "notes": "Description with "quotes""
}

// ✅ Fixed
{
    "notes": "Description with \"quotes\""
}
```

### Symptom: "Missing required key"

**Output:**
```
Error in DAG configuration for my_dag: Missing key 'dataset_name'
```

**Cause**: Required parameter not in config (neither defaults nor DAG entry)

**Required parameters:**
- `dataset_name`
- `dataset_product`
- `data_source`
- `conn_id`
- `write_option`
- `query`

**Fix:**

Add missing parameter to DAG entry:
```json
{
    "dags": [
        {
            "dataset_name": "my_dataset",  // Add this
            "dataset_product": "my_product",
            // ... other parameters
        }
    ]
}
```

Or add to defaults if shared:
```json
{
    "defaults": {
        "dataset_product": "my_product",  // Shared by all DAGs
        // ... other defaults
    },
    "dags": [
        {
            "dataset_name": "dataset_one",
            // dataset_product inherited from defaults
        }
    ]
}
```

### Symptom: Generator completes but file not created

**Possible causes:**

1. **Output directory doesn't exist**: Generator creates it, but check permissions
2. **File path too long**: Rarely an issue on Unix, but possible on Windows
3. **Generator output shows skipped DAG**: Check for errors in that DAG's config

**Fix:**

Check generator output:
```bash
cd utilities
python dag_generator.py

# Look for:
# "Added to git: ../mlgpu/dags/generated/product/dag_name.py"
# If missing, look for error messages above
```

Manually verify:
```bash
ls -la ../mlgpu/dags/generated/product/
```

---

## Issue 3: Data Not Written to Blob Storage

### Symptoms

- DAG runs successfully (all tasks green)
- No data appears in Azure blob storage
- Or data appears in wrong location

### Diagnosis

Check task logs in Airflow UI:

1. Navigate to DAG → Grid view
2. Click on failed/completed task
3. View logs

Look for:
- Azure authentication errors
- Permission denied errors
- Wrong storage account/container

### Common Causes & Fixes

**Cause 1: Storage connection not configured**

**Error in logs:**
```
Connection 'data_agg_bronze' not found
```

**Fix:**
1. Airflow UI → Admin → Connections
2. Add connection with ID `data_agg_bronze`
3. Connection type: Azure Blob Storage
4. Configure storage account credentials

**Cause 2: Wrong storage path in config**

**Check config:**
```json
"storage_account": "dataaggbronze",      // Correct account?
"storage_container": "landing-zone",     // Correct container?
```

**Expected path**: `{container}/{dataset_product}/{data_source}/`

**Verify:**
```bash
# Using Azure CLI
az storage blob list \
  --account-name dataaggbronze \
  --container-name landing-zone \
  --prefix sca_phonehome_kb/
```

**Cause 3: Azure permissions**

**Error in logs:**
```
AuthorizationPermissionMismatch
This request is not authorized to perform this operation
```

**Fix:**
- Verify service principal has Storage Blob Data Contributor role
- Check storage account firewall rules
- Verify connection credentials are current

**Cause 4: Task succeeded but no output**

Check if query returned data:
```python
# In DAG logs, look for query result row count
# Should show: "Query returned N rows"
```

If 0 rows, check:
- Query date window matches execution date
- Source data exists for that date
- Query syntax is correct

---

## Issue 4: Data Pull Query Fails

### Symptoms

- Task `build_data_pull_query` succeeds
- Task `pull_data_from_source` fails
- Logs show SQL error

### Diagnosis

Common SQL errors:

**Syntax Error**
```
ERROR: syntax error at or near "FROM"
```

**Fix**: Test query manually:
```bash
# Replace placeholders with actual dates
psql -h host -U user -d database -c "
SELECT col1, col2
FROM schema.table
WHERE date >= '2024-01-01'
  AND date < '2024-01-02'
"
```

**Table/Column Not Found**
```
ERROR: relation "schema.table" does not exist
ERROR: column "col_name" does not exist
```

**Fix**: 
- Verify table name and schema
- Check column names (case-sensitive in some databases)
- Ensure connection connects to correct database

**Permission Denied**
```
ERROR: permission denied for table table_name
```

**Fix**:
- Verify connection user has SELECT permission
- Check database firewall rules
- Verify connection configuration

**Timeout**
```
ERROR: query timeout exceeded
```

**Fix**:
- Optimize query (add indexes, reduce date range)
- Increase timeout in DAG configuration
- Check database load

### Testing Queries

**Before deploying**, test queries:

1. **Replace placeholders manually:**
```sql
-- Config has:
WHERE daystr >= '{dag_start_date}' AND daystr < '{dag_end_date}'

-- Test with:
WHERE daystr >= '2024-01-01' AND daystr < '2024-01-02'
```

2. **Run against source database:**
```bash
psql -h host -U user -d db -c "SELECT ..."
```

3. **Check row count:**
```sql
-- Add LIMIT during testing
SELECT col1, col2 FROM table WHERE ... LIMIT 10;
```

4. **Verify date columns:**
```sql
-- Check date format
SELECT MIN(date), MAX(date) FROM table;
```

---

## Issue 4.5: PostgreSQL Identifier Quoting Errors

### Symptoms

- Query fails with "relation does not exist" even though table exists
- "syntax error near reserved word" (like `references`, `user`, `order`)
- Query works in MySQL but fails in PostgreSQL
- DAG import fails with Python SyntaxError involving quotes

### Diagnosis

PostgreSQL requires double quotes for case-sensitive identifiers and reserved words. Many errors arise from incorrect quoting.

**Common error patterns:**

**1. Mixed-case table/column not quoted:**
```
ERROR: relation "trimsserver_capec" does not exist
HINT: Perhaps you mean the relation "TRIMSserver_capec"
```

The table is `TRIMSserver_capec` (mixed case) but query used lowercase without quotes.

**2. Reserved word not quoted:**
```
ERROR: syntax error at or near "references"
```

Column named `references` conflicts with SQL keyword.

**3. Using backticks (MySQL syntax):**
```
ERROR: syntax error at or near "`"
```

Backticks `` ` `` are MySQL, not PostgreSQL.

**4. Python SyntaxError with triple quotes:**
```
SyntaxError: unterminated string literal
```

Query ends with `"TRIMSserver_capec"` and uses triple quotes: `f"""..."""` creates `""""`.

### Fix

**For mixed-case identifiers:**

```json
// ❌ WRONG - PostgreSQL folds to lowercase
"query": "SELECT * FROM TRIMSserver_capec"

// ❌ WRONG - Backticks are MySQL syntax
"query": "SELECT * FROM `TRIMSserver_capec`"

// ✅ CORRECT - Double quotes for PostgreSQL
"query": "SELECT * FROM \"TRIMSserver_capec\""
```

**For reserved words:**

```json
// ❌ WRONG - 'references' is SQL keyword
"query": "SELECT id, references FROM table"

// ✅ CORRECT - Quote reserved words
"query": "SELECT id, \"references\" FROM table"
```

**Common reserved words to watch for:**
- `references`, `user`, `order`, `table`, `column`, `index`
- `group`, `where`, `select`, `insert`, `update`, `delete`
- Check [PostgreSQL reserved words list](https://www.postgresql.org/docs/current/sql-keywords-appendix.html)

**For mixed-case columns:**

```json
// ❌ WRONG - base_cwe_confidence gets folded to lowercase
"query": "SELECT base_CWE_confidence FROM table"

// ✅ CORRECT - Quote mixed-case columns
"query": "SELECT \"base_CWE_confidence\" FROM table"
```

**Python string quoting in config:**

The tricky part is quoting for both JSON and SQL:

```json
// Config file (JSON) - escape backslash for JSON, then double-quote for SQL
{
    "query": "SELECT * FROM \"TRIMSserver_capec\""
}
```

In JSON, `\"` becomes `"` in the string, which is what PostgreSQL needs.

**Avoiding triple-quote Python syntax errors:**

If generated DAG uses `f"""..."""`, queries ending with `"` create problems:

```python
# ❌ WRONG - Creates """"
query = f"""SELECT * FROM "TRIMSserver_capec""""

# ✅ FIX 1 - Use single quotes in config, generates single-quoted f-string
# Config: "query": "SELECT * FROM \"Table\""
# Generates: query = f'SELECT * FROM "Table"'

# ✅ FIX 2 - Update template to put closing """ on new line
query = f"""SELECT * FROM "TRIMSserver_capec"
"""
```

### When to Quote

**Always quote in PostgreSQL:**
- Mixed-case: `TRIMSserver_capec`, `base_CWE_confidence`
- Reserved words: `references`, `user`, `order`
- Special characters: `column-name`, `my column`
- Starts with number: `2024_data`

**Don't need to quote:**
- All lowercase with underscores: `trims_vulnerability`
- No reserved words: `scan_count`, `project_id`

**Best practice**: When in doubt, quote it. It doesn't hurt to quote non-reserved lowercase identifiers.

### Testing Queries

Always test against the actual database first:

```bash
# Connect to PostgreSQL
psql -h source-db.example.com -U readonly_user -d analytics

# Check actual table names (case-sensitive)
\dt

# Check actual column names
\d "TRIMSserver_capec"

# Test query
SELECT * FROM "TRIMSserver_capec" LIMIT 1;
```

### Prevention

1. **Use `\d` in psql** to see actual case of names
2. **Quote identifiers in config** using escaped quotes: `\"Name\"`
3. **Test queries before adding to config**
4. **Know your database**: PostgreSQL != MySQL quoting rules
5. **Check for reserved words** in table/column names
6. **Document why you're quoting** in notes field

### Database-Specific Quoting

**PostgreSQL:**
- Use double quotes: `"Identifier"`
- Required for mixed-case and reserved words

**MySQL:**
- Use backticks: `` `identifier` ``
- Or ANSI mode with double quotes

**If migrating between databases**, update all identifier quoting.

---

## Issue 5: Schema Mismatch Errors

### Symptoms

- Data writes to blob successfully
- Unity Catalog processing fails
- Logs show schema mismatch errors

### Common Causes

**New columns in source not in target**

Unity Catalog schema evolution should handle this, but may fail if:
- Column type changed (string → integer)
- Column renamed (old data has old name)

**Fix**:
- Coordinate with data engineering team
- May need manual schema update
- Or adjust query to match existing schema

**Column classification conflicts**

Different runs have different classifications for same column.

**Fix**: Standardize in config:
```json
"column_classification": {
    "email": "confidential"  // Consistent across all runs
}
```

**Partition column mismatch**

If table is partitioned, date columns must align.

**Fix**: Coordinate with data engineering team for partition strategy.

---

## Issue 6: Backfill Not Working

### Symptoms

- DAG enabled with `catchup: true`
- Only current run executes, no historical runs
- Or backfill runs don't process expected dates

### Diagnosis

**Check start_date:**
```python
# In generated DAG
start_date = datetime(2024, 1, 1)

# Backfill runs from start_date to now
```

**Check catchup:**
```python
catchup = True  # Should be True for backfill
```

**Check DAG status in Airflow:**
- Is DAG paused? (Must be unpaused)
- Are there failed runs blocking queue?
- Check max_active_runs limit

### Common Issues

**Issue: start_date in future**

If `start_date > now()`, DAG won't run until that date.

**Fix**: Set start_date to when data is available:
```json
"start_year": 2024,
"start_month": 1,
"start_day": 1
```

**Issue: max_active_runs too low**

With `max_active_runs: 1`, backfill processes one date at a time.

**Fix**: Increase if source can handle parallelism:
```json
"max_active_runs": 5
```

**Issue: Previous failed runs**

Failed runs block the queue.

**Fix**:
1. Airflow UI → DAG → Grid view
2. Clear failed runs
3. Or mark as success if failures can be ignored

**Issue: Source data doesn't exist for backfill dates**

DAG runs but query returns 0 rows for historical dates.

**Fix**:
- Verify source has historical data
- Adjust start_date to when data begins
- Or accept 0-row runs (they'll succeed, just won't load data)

---

## Issue 7: Airflow Connection Not Found

### Symptom

```
ERROR: Connection 'conn_id_name' not found
```

### Fix

Create connection in Airflow UI:

1. **Navigate**: Admin → Connections → Add
2. **Fields:**
   - Connection Id: `conn_id_name` (must match config exactly)
   - Connection Type: Choose based on database (PostgreSQL, MySQL, etc.)
   - Host: Database host
   - Schema: Database name
   - Login: Username
   - Password: Password
   - Port: Database port (usually 5432 for PostgreSQL, 3306 for MySQL)

3. **Test Connection** (if available)
4. **Save**

**Naming convention**: Use `_ro` suffix for read-only connections:
```json
"conn_id": "kb_analytics_blackduck_ro"
```

---

## Issue 8: Generated DAG Has Syntax Error

### Symptom

DAG doesn't appear in Airflow UI, or shows import error.

### Diagnosis

Check Airflow logs for import errors:
```bash
airflow dags list-import-errors
```

Or test Python syntax:
```bash
python ../mlgpu/dags/generated/product/dag_name.py
```

### Common Causes

**Unescaped quotes in config**

Config:
```json
"notes": "Description with "quotes""
```

Generated:
```python
notes = "Description with "quotes""  # ❌ Syntax error
```

**Fix**: Escape quotes in JSON:
```json
"notes": "Description with \"quotes\""
```

**Newlines in query string**

Config:
```json
"query": "SELECT col1,
col2 FROM table"
```

Generated:
```python
query = "SELECT col1,
col2 FROM table"  # ❌ Syntax error (unescaped newline)
```

**Fix**: Use single-line string or escape newlines:
```json
"query": "SELECT col1, col2 FROM table"
```

**Special characters in values**

Backslashes, quotes, etc. in values must be escaped.

**Fix**: Use proper JSON encoding for all string values.

---

## Issue 9: DAG Runs But No Data in Unity Catalog

### Symptoms

- DAG succeeds
- Data in blob storage
- Not showing up in Unity Catalog

### Diagnosis

This is usually a downstream processing issue, not a DAG issue.

**Check:**
1. Processing backlog table - is dataset registered?
2. Conformance pipeline - is it running?
3. Unity Catalog - check for processing errors

**Backlog entry:**

The DAG inserts a record in the processing backlog:
```python
# In set_dataset_params task
"backlog_status": "pending"
```

Verify this entry exists and has status "pending" or "processing".

**Conformance pipeline:**

Separate process reads backlog and processes data to Unity Catalog.

**Fix**:
- Check with data engineering team
- Verify conformance pipeline is running
- Check conformance pipeline logs for errors
- May need to reprocess manually

---

## Issue 10: Slow Backfill Performance

### Symptoms

- Backfill taking very long time
- Many days/months to catch up
- Source database under heavy load

### Optimization Strategies

**1. Increase parallelism (if source can handle it):**
```json
"max_active_runs": 5  // From default 3
```

**2. Optimize query:**
- Add indexes on date columns in source
- Select only needed columns (not SELECT *)
- Filter unnecessary rows

**3. Partition backfill:**

Instead of backfilling all at once, do it in phases:
```json
// Phase 1: Recent 90 days
"start_year": 2024,
"start_month": 3,
"start_day": 1,
"catchup": true

// After Phase 1 completes, update config for Phase 2:
"start_year": 2024,
"start_month": 1,
"start_day": 1
```

**4. Coordinate with DBAs:**
- Schedule backfill during off-peak hours
- May need to adjust source database resources
- Consider read replica for backfill

**5. Monitor progress:**
```bash
# Check completed runs in Airflow UI
# Or query Airflow metadata database
```

---

## Debugging Checklist

When troubleshooting a DAG issue, work through this checklist:

### Configuration

- [ ] Config file is valid JSON
- [ ] All required parameters present
- [ ] merge_on formatted correctly (if needed)
- [ ] Query has correct placeholders
- [ ] Connection ID matches Airflow connection

### Generation

- [ ] Generator ran successfully
- [ ] Generated file exists at expected path
- [ ] Generated file has correct parameter values
- [ ] No Python syntax errors in generated file

### Airflow

- [ ] DAG appears in Airflow UI
- [ ] DAG is unpaused
- [ ] Connection exists and is configured
- [ ] No import errors

### Execution

- [ ] Query runs successfully against source
- [ ] Data returned from query
- [ ] Data written to blob storage
- [ ] Correct blob storage path
- [ ] Processing backlog entry created

### Downstream

- [ ] Conformance pipeline running
- [ ] Backlog entry being processed
- [ ] Data appearing in Unity Catalog

---

## Getting Help

**Before asking for help, gather:**

1. Config file content
2. Generated DAG file (or relevant excerpts)
3. Error messages from Airflow logs
4. DAG run ID and execution date
5. What you've tried so far

**Where to ask:**
- Data engineering Slack channel
- Team wiki/documentation
- Senior data engineers
- This troubleshooting guide!

**When escalating:**
- Provide context (what you're trying to do)
- Share error messages (full text, not just excerpt)
- Explain what you've tried
- Include relevant config/code snippets
