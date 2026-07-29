---
description: Black Duck data product analytics — catalog structure, Spark SQL patterns, common gotchas, query standards
disable-model-invocation: false
---

# Black Duck Data Product Analytics

## Architecture Overview

Three catalog layers, each with multiple schemas:

| Catalog | Schemas | Purpose |
|---------|---------|---------|
| `data_product_marts` | `core`, `enterprise`, product-specific | Analytics-ready; **start here** |
| `data_product_intermediate` | `core`, `enterprise`, product-specific | Shared calculations used across marts |
| `data_product_staging` | product-specific only | Raw product-native data |

### Schema Types

- **`core`** (marts + intermediate): Cross-product entities — customers, applications, findings, scans, targets
- **`enterprise`** (marts + intermediate): Cross-product analytics
- **Product-specific** (all catalogs): Deep dives — `llm_gateway`, `polaris`, `cd`, etc.

**Default starting point**: `data_product_marts.core`

## Scale Context

- ~4K customers → ~130K applications → ~317K targets → ~2.8M findings → ~20M scans

## Common Gotchas

- Polaris EU/US customers may have duplicate records — deduplicate on a stable key
- Customer names in Polaris may be MD5 hashes rather than readable names
- Not all foreign key relationships are clean (orphaned records exist)
- Scan volumes vary dramatically by product/tool type — avoid raw cross-tool comparisons
- `model_group`, `proxy_base_url`, `team_alias` in `llm_gateway` tables are VARIANT — cast to STRING before grouping: `CAST(model_group AS STRING)`

## Analysis Workflow

1. **Explore**: `mcp__superduck__data_products_list_data_products()` — discover available tables
2. **Understand**: `mcp__superduck__data_products_describe_data_product(full_table_name)` — schema + known issues
3. **Query**: Use Spark SQL with full three-part names: `catalog.schema.table`
4. **Validate**: Cross-check against known data quality limitations
5. **Document**: Note data quality context in findings

## SQL Standards

- Full three-part table names always: `data_product_marts.core.customers`
- Backtick-escape special characters: `` `special-table` ``
- Explicit column lists — avoid `SELECT *` on large tables
- `LIMIT` on all exploratory queries
- Filter on partition columns (dates, product, region) when available — reduces scan cost
- Default timeout: 60s, increase to 120-300 for complex joins
- Default limit: 100 rows
- Add comments on non-obvious filters

## Query Patterns

### Core Joins

```sql
-- Inner join with soft-delete filtering on both sides
SELECT c.id, c.name, COUNT(DISTINCT a.id) AS app_count
FROM data_product_marts.core.customers c
JOIN data_product_marts.core.applications a ON c.id = a.customer_id
WHERE c.is_deleted = false AND a.is_deleted = false
GROUP BY c.id, c.name
ORDER BY app_count DESC LIMIT 20;

-- Left join preserving nulls
SELECT c.id, c.name, COALESCE(COUNT(DISTINCT a.id), 0) AS app_count
FROM data_product_marts.core.customers c
LEFT JOIN data_product_marts.core.applications a
  ON c.id = a.customer_id AND a.is_deleted = false
WHERE c.is_deleted = false
GROUP BY c.id, c.name
ORDER BY app_count DESC LIMIT 20;

-- Multi-level traversal: customer → application → target → finding
SELECT c.id, c.name,
       COUNT(DISTINCT a.id) AS app_count,
       COUNT(DISTINCT t.id) AS target_count,
       COUNT(DISTINCT f.id) AS finding_count
FROM data_product_marts.core.customers c
LEFT JOIN data_product_marts.core.applications a
  ON c.id = a.customer_id AND a.is_deleted = false
LEFT JOIN data_product_marts.core.targets t
  ON a.id = t.application_id AND t.is_deleted = false
LEFT JOIN data_product_marts.core.findings f
  ON t.id = f.target_id AND f.is_deleted = false
WHERE c.is_deleted = false
GROUP BY c.id, c.name
ORDER BY finding_count DESC LIMIT 20;
```

### Aggregation

```sql
-- Distribution with percentage
SELECT severity, COUNT(*) AS count,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM data_product_marts.core.findings
WHERE finding_status = 'open'
GROUP BY severity ORDER BY count DESC;

-- Grouped with HAVING filter
SELECT customer_id, COUNT(*) AS finding_count
FROM data_product_marts.core.findings
WHERE is_deleted = false AND severity = 'critical'
GROUP BY customer_id
HAVING COUNT(*) > 10
ORDER BY finding_count DESC;

-- Multiple aggregations per group
SELECT customer_id,
       COUNT(*) AS total_findings,
       COUNT(DISTINCT target_id) AS affected_targets,
       MIN(first_detected_at) AS earliest,
       MAX(first_detected_at) AS latest
FROM data_product_marts.core.findings
WHERE is_deleted = false
GROUP BY customer_id
ORDER BY total_findings DESC LIMIT 100;
```

### Time Series

```sql
-- Monthly aggregation
SELECT DATE_TRUNC('month', created_at) AS month,
       COUNT(*) AS finding_count
FROM data_product_marts.core.findings
WHERE is_deleted = false
  AND created_at >= '2024-01-01' AND created_at < '2025-01-01'
GROUP BY month ORDER BY month;

-- Cumulative running total
SELECT DATE_TRUNC('month', created_at) AS month,
       COUNT(*) AS new_count,
       SUM(COUNT(*)) OVER (ORDER BY DATE_TRUNC('month', created_at)) AS cumulative
FROM data_product_marts.core.findings
WHERE is_deleted = false
GROUP BY month ORDER BY month;

-- Week-over-week change
WITH weekly AS (
    SELECT DATE_TRUNC('week', created_at) AS week, COUNT(*) AS count
    FROM data_product_marts.core.findings
    WHERE is_deleted = false
    GROUP BY week
)
SELECT week, count,
       LAG(count) OVER (ORDER BY week) AS prev_week,
       count - LAG(count) OVER (ORDER BY week) AS change,
       ROUND((count - LAG(count) OVER (ORDER BY week)) * 100.0
             / NULLIF(LAG(count) OVER (ORDER BY week), 0), 2) AS pct_change
FROM weekly ORDER BY week DESC;

-- Scan volume by tool type (monthly)
SELECT DATE_TRUNC('month', created_at) AS month,
       tool_type, COUNT(*) AS scan_count
FROM data_product_marts.core.scans
WHERE created_at >= DATE_SUB(CURRENT_DATE(), 365)
GROUP BY 1, 2 ORDER BY 1, 2;

-- LLM Gateway weekly (VARIANT cast)
SELECT CAST(model_group AS STRING) AS model,
       COUNT(*) AS requests,
       SUM(total_tokens) AS total_tokens
FROM data_product_staging.llm_gateway.stg_llm_gateway__llm_gateway_usage
WHERE created_at >= DATE_SUB(CURRENT_DATE(), 30)
GROUP BY CAST(model_group AS STRING)
ORDER BY requests DESC;
```

### Window Functions

```sql
-- Top-N ranking
SELECT id, name, score,
       RANK() OVER (ORDER BY score DESC) AS rank
FROM data_product_marts.core.targets
WHERE is_deleted = false
ORDER BY rank LIMIT 10;

-- Top-N per partition (5 most recent findings per customer)
SELECT customer_id, id, severity,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY first_detected_at DESC) AS rank
FROM data_product_marts.core.findings
WHERE is_deleted = false
QUALIFY rank <= 5;
```

### CTEs for Complex Queries

```sql
WITH active_findings AS (
    SELECT * FROM data_product_marts.core.findings
    WHERE is_deleted = false AND severity IN ('critical', 'high')
)
SELECT customer_id, COUNT(*) AS critical_high_count
FROM active_findings
GROUP BY customer_id
ORDER BY critical_high_count DESC LIMIT 100;

-- Multi-CTE for layered analysis
WITH customer_apps AS (
    SELECT customer_id, COUNT(*) AS app_count
    FROM data_product_marts.core.applications
    WHERE is_deleted = false GROUP BY customer_id
),
customer_findings AS (
    SELECT customer_id, COUNT(*) AS finding_count
    FROM data_product_marts.core.findings
    WHERE is_deleted = false GROUP BY customer_id
)
SELECT c.id, c.name,
       COALESCE(ca.app_count, 0) AS applications,
       COALESCE(cf.finding_count, 0) AS findings
FROM data_product_marts.core.customers c
LEFT JOIN customer_apps ca ON c.id = ca.customer_id
LEFT JOIN customer_findings cf ON c.id = cf.customer_id
WHERE c.is_deleted = false
ORDER BY findings DESC LIMIT 100;
```

### Data Quality Checks

```sql
-- Find duplicate primary keys
SELECT id, COUNT(*) AS dupes
FROM data_product_marts.core.customers
WHERE is_deleted = false
GROUP BY id HAVING COUNT(*) > 1;

-- Orphaned records (findings with no matching target)
SELECT f.*
FROM data_product_marts.core.findings f
LEFT JOIN data_product_marts.core.targets t ON f.target_id = t.id
WHERE f.is_deleted = false AND t.id IS NULL
LIMIT 100;

-- NULL analysis
SELECT COUNT(*) AS total,
       COUNT(severity) AS non_null,
       COUNT(*) - COUNT(severity) AS null_count,
       ROUND((COUNT(*) - COUNT(severity)) * 100.0 / COUNT(*), 2) AS null_pct
FROM data_product_marts.core.findings
WHERE is_deleted = false;
```

## Filtering Patterns

```sql
-- Multiple values
WHERE severity IN ('critical', 'high')

-- Date range (prefer range over function wrapping)
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'  -- correct
WHERE DATE(created_at) = '2024-01-01'  -- avoid: prevents partition pruning

-- Pattern matching
WHERE name ILIKE '%search%'

-- NULL defaults
SELECT COALESCE(severity, 'unknown') AS severity
```

## Reporting Standards

- Always state the methodology and table sources
- Note data quality limitations that may affect conclusions
- Specify the time period and data freshness
- Suggest follow-up analyses when relevant
- For large result sets: use Agent subagent to avoid context overflow
