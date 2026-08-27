# Validation And Quality Checks

Use this reference after the first substantive edit and again before finishing.

## Focused Validation Order

Prefer the narrowest check that can falsify the change:

1. run the impacted model(s) with dependencies
2. run tests for the impacted model(s)
3. run parse/compile sanity if the change is broader
4. run repo quality scripts if naming/docs changed
5. run pre-commit if the change is ready for review

## Core dbt Commands

Run from the dbt project directory when possible.

```bash
dbt parse --target dev

dbt run --target dev --select +<model_or_selector>

dbt test --target dev --select <model_or_selector>
```

Useful selector patterns:

```bash
# upstream dependencies + model
dbt run --target dev --select +int_customer_unification

# model + downstream dependents
dbt run --target dev --select int_customer_unification+

# upstream and downstream
dbt run --target dev --select +int_customer_unification+
```

## Layer-Scoped Commands

```bash
dbt run --target dev --select staging.*
dbt run --target dev --select intermediate.*
dbt run --target dev --select marts.*
```

## Repo Quality Scripts

Use when the change affects naming, documentation, or catalog metadata coverage.

```bash
python uc_transforms/scripts/check_naming_conventions.py
python uc_transforms/scripts/check_yaml_coverage.py
```

Other available checks in this repo may also be relevant depending on the change:

```bash
python uc_transforms/scripts/check_catalog_descriptions.py
python uc_transforms/scripts/check_masking_coverage.py
```

## Pre-Commit

If hooks are installed and the change is ready for review:

```bash
pre-commit run --all-files
```

## What To Verify Manually

After commands pass, confirm:

- model names still follow `stg_`, `int_`, and marts domain conventions
- YAML exists for new models or sources
- model descriptions reflect actual semantics
- NULL-vs-zero handling matches the source contract
- dropped columns were intentional and documented

## Common Failure Patterns

- missing `dbt_project.yml` routing when adding a new product-specific layer folder
- undocumented new columns causing YAML coverage failures
- semantic regressions from `coalesce()` applied too early
- repeated SQL fragments left inline when a macro would reduce risk and drift
