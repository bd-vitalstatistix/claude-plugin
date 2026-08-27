---
name: dbt-uc-data-products
description: Add new data products and modify existing data products in the dbt-uc-transforms project. Use this whenever a user asks to add a new product/source, create staging/intermediate/marts models, update source YAML docs, adjust null semantics, add product-specific macros, or extend existing product models in Unity Catalog. Also use this for requests that mention sca_phonehome_kb, sast_telemetry, data product onboarding, model naming, YAML documentation coverage, or dbt layer changes.
---

# dbt UC Data Products

Add new data products and safely modify existing data products in dbt-uc-transforms using the project's staging -> intermediate -> marts architecture and Unity Catalog conventions.

Use the bundled references to keep the main workflow concise:

- Read `references/workflows.md` to choose the correct add / modify / extend path.
- Read `references/templates.md` when you need file shapes for staging, intermediate, macros, or `dbt_project.yml` wiring.
- Read `references/validation.md` immediately after the first substantive edit and again before finishing.

## What This Skill Covers

- Adding a new product dataset in staging
- Extending a product into intermediate and marts
- Updating existing product models without breaking downstream usage
- Applying documentation standards so SuperDuck can read model metadata
- Running the right validation commands before commit

## Sources This Skill Is Based On

- Project architecture and workflow guidance from confluence_instructions_temp.md
- Repository setup and standards from README.md and uc_transforms/README.md
- Real change patterns from:
  - 83a7a3305662115cb0e8f5cecab007934003b4ae (sast_telemetry intermediate expansion)
  - d8d48577edf24763764253d7c300ab31a28e3673 (sca_phonehome_kb staging onboarding)

## Start Here

Before editing:

1. Choose the change type in `references/workflows.md`.
2. Gather product name, source tables, grain, target layer, and critical semantics.
3. If the request is ambiguous on layer or semantics, clarify before writing SQL.

Use `references/templates.md` for the expected file patterns. The two key commit-backed patterns are:

- `sca_phonehome_kb`: source YAML plus staging SQL with explicit casts, renamed columns, and preserved NULL semantics
- `sast_telemetry`: product-specific intermediate models, YAML docs, `dbt_project.yml` routing, and extracted macros when repetition appears

## Documentation Rules (Important)

Follow repo standards from uc_transforms/README.md:

- Model descriptions should explain data meaning, grain, joins, caveats, and nullable behavior
- Avoid pipeline implementation details in model docs when they are not needed by analysts
- Capture material semantic caveats (for example version-gated NULL behavior) where users will read them
- Keep docs updated so catalog comments remain accurate in Unity Catalog

Why this matters:

- `persist_docs` plus post-hook catalog updates make YAML descriptions visible in downstream tools
- Missing or weak YAML directly degrades discoverability and trust

## Modification Rules

When changing existing models:

1. Confirm current grain and key columns before editing.
2. Keep stable downstream-facing contracts unless the change explicitly requires a break.
3. Update SQL and YAML together when semantics change.
4. If dropping a column, verify it is unused or truly dead and explain why.
5. Use `references/validation.md` for the first focused check immediately after editing.

Use commit-style reasoning from real examples:

- Semantic fix: preserve NULL meaning instead of flattening with COALESCE(0)
- Dead column cleanup: remove only after validating persistent non-population
- Unverified fields: label as unverified instead of inventing certainty

## Validation

Use `references/validation.md` for exact command order and repo quality checks.

At minimum:

- run the narrowest impacted `dbt run`
- run the impacted `dbt test`
- run broader parse/quality checks when naming or docs changed

## Output Format For This Skill

When responding to a user request, structure output as:

1. Change type: add / modify / extend
2. Files to create or update (exact paths)
3. SQL and YAML changes to apply
4. Validation commands to run
5. Risks or assumptions

Keep it concrete and file-oriented.

## Fast Decision Heuristics

- New source table only: start in staging
- Reusable business logic across products: intermediate core/enterprise
- Product-specific transformations: intermediate <product>
- End-user analytics tables: marts
- Repeated pivot/list expressions: macro

## Guardrails

- Do not push changes directly to prod targets while developing
- Do not change naming conventions (`stg_`, `int_`, marts domain prefixes)
- Do not collapse unknown semantics into defaults without explicit sign-off
- Do not leave undocumented columns in new models

## Example User Requests That Should Trigger This Skill

- Add a new data product under staging for a new telemetry source
- Create intermediate models for an existing product and wire dbt_project.yml
- Update an existing staging model to reflect corrected NULL semantics
- Add docs for new columns so they appear in Unity Catalog descriptions
- Extend sca_phonehome_kb style onboarding for another source
