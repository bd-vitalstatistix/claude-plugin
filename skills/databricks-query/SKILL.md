---
description: Execute SQL queries against Databricks and retrieve actual results via the SQL Statement Execution API
disable-model-invocation: false
---

# Databricks Query Execution

## The Only Correct Approach

**Never use** `databricks jobs submit` or `databricks jobs get-run-output` to retrieve data — these return job execution metadata (status, timing), not query results. Notebooks and job APIs are for monitoring, not data extraction.

**Always use** the SQL Statement Execution API 2.0.

## Workspace Config

| Profile | Warehouse ID | Default Catalog |
|---------|--------------|-----------------|
| `blackduck_general` | `e3c0f93b080b8c46` | `data_product_marts` |

Always specify `--profile blackduck_general` to be explicit about which workspace you're targeting.

## Synchronous Query (preferred)

```bash
databricks --profile blackduck_general api post /api/2.0/sql/statements \
  --json '{
    "warehouse_id": "e3c0f93b080b8c46",
    "statement": "SELECT * FROM data_product_marts.core.customers LIMIT 10",
    "wait_timeout": "50s",
    "format": "JSON_ARRAY"
  }'
```

`wait_timeout: "50s"` is the safe cross-instance maximum — values above 50s are rejected on some instances. The response includes results directly when the query completes within the timeout.

## Async Query (for long-running queries)

```bash
# Submit
RESPONSE=$(databricks --profile blackduck_general api post /api/2.0/sql/statements \
  --json '{
    "warehouse_id": "e3c0f93b080b8c46",
    "statement": "SELECT ...",
    "wait_timeout": "0s",
    "format": "JSON_ARRAY"
  }')

STATEMENT_ID=$(echo $RESPONSE | python3 -c "import json,sys; print(json.load(sys.stdin)['statement_id'])")

# Poll
databricks --profile blackduck_general api get /api/2.0/sql/statements/$STATEMENT_ID
```

Use `wait_timeout: "0s"` to get a statement ID immediately, then poll until `status.state == "SUCCEEDED"`.

## Result Format

```json
{
  "manifest": {
    "schema": {
      "columns": [{"name": "customer_name", "type_name": "STRING"}, ...]
    },
    "total_row_count": 10
  },
  "result": {
    "data_array": [["Amazon.com, Inc.", ...], ...]
  },
  "status": {"state": "SUCCEEDED"}
}
```

Parse results with `python3 -c "import json,sys; d=json.load(sys.stdin); ..."` — avoid `jq` for nested structures on macOS.

## SQL Formatting Rules

- No literal newlines in JSON string values — use `\n` escape sequences if needed
- Use full three-part names: `catalog.schema.table`
- Default to `LIMIT 100` unless explicitly aggregating

## Authentication Issues

If you see "Refresh token is invalid":
```bash
rm -rf ~/.databricks/token-cache.json
databricks auth login --profile blackduck_general
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Missing `--profile` | Always pass `--profile blackduck_general` |
| `wait_timeout` > 50s | Use `"50s"` as the ceiling |
| Literal newlines in SQL string | Use `\n` escape sequences — bare newlines in JSON break the request |
| Using job API for results | Switch to SQL Statement Execution API |
