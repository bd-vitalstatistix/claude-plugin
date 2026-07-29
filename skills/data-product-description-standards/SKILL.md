---
description: Standards for writing dbt model and column descriptions that propagate to SuperDuck — grain, relationships, caveats, and what not to document
disable-model-invocation: false
---

# Data Product Description Standards

Model descriptions in `_<schema>__models.yml` files must describe **the data and its relationships to other models only**. These descriptions become the authoritative source of truth in SuperDuck's `describe_data_product` and `list_data_products` tools.

## What belongs in a description

- **Grain**: What one row represents (e.g. "one row per application version")
- **Relationships**: Which other models to join on, and which keys to use
- **Data quality caveats**: When nullable columns are populated vs null, known gaps
- **Cross-product join patterns**: How a staging model links into `marts.core`

## What does not belong

- Source system internals (ORM models, source DB table names, framework behaviour)
- Pipeline mechanics (Airflow DAG names, catchup settings, ingestion cadence)
- Implementation details (cast operations, rename rationale, backfill history)
- Row counts or cardinality estimates that will go stale

## Examples

**Wrong** — describes the implementation, not the data:
```yaml
- name: stg_trims__advisory_history
  description: >
    Full Django simple-history audit log for every field change to every BDSA (advisory).
    Backfilled via catchup=true in the Airflow DAG. One row per advisory version change.
```

**Right** — describes the data and its relationships:
```yaml
- name: stg_trims__advisory_history
  description: >
    Complete change history for all BDSAs. One row per advisory version.
    Use this for point-in-time or lifecycle queries; for current state only use stg_trims__advisories.
    Links to stg_trims__users via history_user_id and via created_by_username / edited_by_username.
```

## Column descriptions

Column descriptions follow the same rule. In addition:

- If a column is a foreign key, say which model and column to join to
- For categorical/enum columns, list known values sourced from actual data — don't infer from code
- If a column has a non-obvious format or domain, describe the values, not the source field name
- Omit descriptions for self-evident columns (timestamps named `created_at`, boolean flags named `is_active`)

**Wrong:**
```yaml
- name: weakness_id
  description: BD weakness identifier (cast to string, e.g. '152')
```

**Right:**
```yaml
- name: weakness_id
  description: BD weakness identifier — join to marts.core.findings via finding_type_id = 'w-' || weakness_id
```

## How descriptions reach SuperDuck

Writing column descriptions in `_<product>__models.yml` is the mechanism that makes them visible in SuperDuck MCP tools. The flow:

1. Descriptions are written in `_<product>__models.yml`
2. `persist_docs: { relation: true, columns: true }` is configured globally in `dbt_project.yml`
3. On every production dbt run, `update_catalog_descriptions()` applies them to Unity Catalog as `ALTER COLUMN ... COMMENT` statements
4. SuperDuck MCP tools read those comments directly from Unity Catalog

No separate registration step is needed — the YAML is the source of truth.

## Rationale

Descriptions that reference implementation details rot immediately: DAG names change, frameworks are replaced, source tables are renamed. Data relationships are stable. A reader using the data catalogue should understand what the table contains and how to join it without knowing anything about the ingestion stack.
