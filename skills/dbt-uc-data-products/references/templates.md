# Templates And File Patterns

Use this reference when you need the expected file shapes for staging, intermediate, or dbt project wiring.

## Staging SQL Pattern

```sql
with source as (
    select * from {{ source('<product>', '<table>') }}
),

renamed as (
    select
        cast(raw_id as string) as entity_id,
        cast(raw_date as date) as snapshot_date,
        cast(raw_timestamp as timestamp) as reported_at,
        cast(raw_metric as bigint) as metric_count,
        '<product>' as data_source
    from source
)

select * from renamed
```

Use this pattern when the goal is source-aligned cleanup:

- keep the first CTE as a simple `source()` read
- do renames and casts in a second CTE
- keep names analyst-friendly
- add `data_source`
- avoid hiding important source semantics with premature defaults

## Source YAML Pattern

```yaml
version: 2

sources:
  - name: <product>
    description: >
      Source-aligned data for <product>. Document the grain and any major usage caveats.
    database: <catalog>
    schema: <schema>
    tables:
      - name: <table>
        description: >
          One row per <grain>. Document important caveats such as NULL semantics,
          version gates, or identity limitations.
        columns:
          - name: raw_id
            description: Source identifier for the entity.
          - name: raw_date
            description: Date associated with the record.
```

Prefer documenting:

- grain
- identity caveats
- NULL behavior
- known deprecations
- join limitations

## Intermediate SQL Naming Pattern

Create product-specific intermediate SQL files like:

- `int_<product>__customers.sql`
- `int_<product>__latest_scan_per_project.sql`
- `int_<product>__all_scans.sql`

Choose names that describe the business entity or selection logic, not implementation details.

## Intermediate YAML Pattern

```yaml
version: 2

models:
  - name: int_<product>__<entity>
    description: >
      One row per <grain>. Explain what this model adds or normalizes and how it
      should be joined downstream.
    columns:
      - name: entity_id
        description: Unique identifier for the entity.
        tests:
          - not_null
      - name: customer_id
        description: Customer identifier used for downstream joins.
```

Prefer model descriptions that explain:

- final grain
- intended joins
- semantics of nullable columns
- important filters or latest-record logic

## dbt_project.yml Pattern For Product-Specific Intermediate

```yaml
models:
  uc_transforms:
    intermediate:
      <product>:
        +schema: "{{ '<product>' if target.name in ['prod', 'databricks_job'] else target.schema }}"
        +meta:
          layer: "intermediate"
          domain: "<product>"
```

Use this when a new product-specific intermediate folder is introduced.

## Macro Extraction Pattern

Create `macros/<product>.sql` when the same pivot lists, enumerations, or column maps repeat across multiple models.

Typical use cases:

- pivot language lists
- pivot extension lists
- repeated field maps
- generated select fragments reused across models

## Documentation Standard

Descriptions should focus on data meaning and downstream use, not pipeline mechanics.

Good description content:

- one row per what
- how to join it
- what caveats matter for interpretation
- when nullable fields are expected

Avoid leading with:

- ingestion job names
- raw ORM details
- cast implementation notes
- incidental development history
