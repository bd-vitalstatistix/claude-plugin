# Workflows

Use this reference when the user asks to add a new data product, modify an existing one, or extend a product into another dbt layer.

## Choose The Path

Pick one path before editing:

1. Add a new product or dataset
2. Modify an existing product model
3. Extend an existing product into another layer

If the prompt is ambiguous, clarify the intended layer:
- staging only
- staging + intermediate
- full path to marts

## Required Inputs

Collect these before writing code:

- product or source name
- source table names
- expected grain
- target layer
- important semantic caveats, especially NULL vs zero
- expected downstream consumers or joins

If any of these are missing and the omission affects model shape or semantics, ask first.

## Workflow: Add A New Staging Product

Use this for requests like "add a new telemetry source" or "onboard a new data product".

Typical files:

- `models/staging/<product>/_<product>__sources.yml`
- `models/staging/<product>/stg_<product>__<entity>.sql`
- optionally `models/staging/<product>/_<product>__models.yml`
- occasionally `dbt_project.yml` if layer routing/config is missing

Implementation pattern:

1. Create a source YAML with table grain and column descriptions.
2. Create the staging SQL with a `source` CTE.
3. Rename raw fields to analytics-friendly names.
4. Cast to explicit types.
5. Add a literal `data_source` column.
6. Preserve meaningful source semantics, especially NULL vs zero.
7. Add or update tests/docs as needed.

This pattern follows the `sca_phonehome_kb` staging onboarding commit.

## Workflow: Add Product-Specific Intermediate Models

Use this for requests like "create reusable product logic" or "add intermediate tables for product X".

Typical files:

- `models/intermediate/<product>/int_<product>__*.sql`
- `models/intermediate/<product>/_int_<product>__models.yml`
- `dbt_project.yml`
- `macros/<product>.sql` when repeated SQL fragments appear across models

Implementation pattern:

1. Confirm whether logic belongs in product-specific intermediate, `core`, or `enterprise`.
2. Add SQL models under `models/intermediate/<product>/`.
3. Add dbt project config for schema routing and metadata.
4. Add YAML docs for all exposed columns.
5. Extract repeated pivot or list logic into a product macro when duplication is obvious.

This pattern follows the `sast_telemetry` intermediate expansion commit.

## Workflow: Modify Existing Models

Use this when the user wants to change semantics, add/drop columns, or correct documentation.

1. Confirm current grain and join keys.
2. Preserve stable downstream-facing contracts unless the change explicitly requires a break.
3. Update SQL and YAML together when semantics change.
4. If removing a column, verify it is dead or intentionally deprecated.
5. If semantics are uncertain, mark them as unverified rather than inventing certainty.

## Semantics Rules

Treat these as high-priority review points:

- NULL and zero are not interchangeable unless the source contract says so.
- Version-gated fields need explicit documentation.
- Inferred columns should be marked as inferred or unverified when evidence is incomplete.
- Avoid baking ingestion or infrastructure details into analyst-facing model descriptions unless they materially affect interpretation.

## Layer Selection Heuristics

- new source-aligned table: staging
- reusable business logic across products: intermediate `core` or `enterprise`
- product-specific reusable logic: intermediate `<product>`
- analytics-ready output for business users: marts
- repeated SQL enumerations or pivots: macro

## Output Shape

When using this skill, respond in this order:

1. change type: add / modify / extend
2. exact files to create or update
3. SQL/YAML/dbt config changes to make
4. validation commands
5. risks, assumptions, or open semantic questions
