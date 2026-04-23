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

- Polaris EU/US customers may have duplicate records
- Customer names in Polaris may be MD5 hashes
- Not all foreign key relationships are clean (orphaned records exist)
- Scan volumes vary dramatically by product/tool type
- `model_group`, `proxy_base_url`, `team_alias` in `llm_gateway` tables are VARIANT — cast to STRING before grouping

## Analysis Workflow

1. **Explore**: `mcp__service-mcp__data_products_list_data_products()` — list available tables
2. **Understand**: `mcp__service-mcp__data_products_describe_data_product(full_table_name)` — schema + known issues
3. **Query**: Use Spark SQL with three-part names: `catalog.schema.table`
4. **Validate**: Cross-check against known data quality limitations
5. **Document**: Note data quality context in findings

## SQL Standards

- Always use full three-part table names: `data_product_marts.core.customers`
- Default timeout: 60 seconds; increase for complex queries
- Default limit: 100 rows; adjust as needed
- Before using example queries, verify current column names via `describe_data_product()`

## Common Query Patterns

```sql
-- Customer overview
SELECT customer_name, COUNT(DISTINCT application_id) AS app_count
FROM data_product_marts.core.customers c
JOIN data_product_marts.core.applications a ON c.customer_id = a.customer_id
GROUP BY customer_name LIMIT 20;

-- Security posture by severity
SELECT severity, COUNT(*) AS finding_count
FROM data_product_marts.core.findings
WHERE finding_status = 'open'
GROUP BY severity ORDER BY finding_count DESC;

-- Scan volume trends
SELECT tool_type, DATE_TRUNC('month', created_at) AS month, COUNT(*) AS scan_count
FROM data_product_marts.core.scans
WHERE created_at >= DATE_SUB(CURRENT_DATE(), 365)
GROUP BY tool_type, month ORDER BY month DESC, scan_count DESC;

-- LLM Gateway weekly summary (VARIANT cast required)
SELECT COUNT(*) AS total_requests,
       COUNT(DISTINCT user_id) AS unique_users,
       SUM(total_tokens) AS total_tokens,
       ROUND(SUM(response_cost), 2) AS cost_usd
FROM data_product_staging.llm_gateway.stg_llm_gateway__llm_gateway_usage
WHERE created_at >= 'YYYY-MM-DD' AND created_at < 'YYYY-MM-DD';

-- LLM Gateway by model (VARIANT cast)
SELECT CAST(model_group AS STRING) AS model, COUNT(*) AS requests
FROM data_product_staging.llm_gateway.stg_llm_gateway__llm_gateway_usage
WHERE created_at >= 'YYYY-MM-DD'
GROUP BY CAST(model_group AS STRING) ORDER BY requests DESC LIMIT 10;
```

## Reporting Standards

- Always state the methodology and table sources
- Note data quality limitations that may affect conclusions
- Specify the time period and data freshness
- Suggest follow-up analyses when relevant
- For large result sets: use Agent subagent to avoid context overflow
