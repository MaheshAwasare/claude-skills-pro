---
name: dbt-data-modeling
description: Build a maintainable dbt project — staging/intermediate/mart layering, sources/refs, tests, macros, incremental models, snapshots, and the discipline that prevents the "1500 models, 0 tests" antipattern. Use when starting a new dbt project or refactoring one that's grown into a hairball.
---

# dbt Data Modeling

dbt's docs cover features. This skill covers *layout and discipline* — the parts that decide whether your warehouse stays understandable past the first 50 models.

## When to use

- New dbt project (Snowflake / BigQuery / Redshift / Postgres / DuckDB).
- Refactoring a project that's grown without layering.
- Setting up CI for dbt for the first time.

## When NOT to use

- Pure transformation jobs that don't need lineage (one-off Python scripts).
- Streaming pipelines — dbt is batch-first; use Materialize or RisingWave.

## The layering (the only thing that matters)

```
models/
  staging/       # 1:1 with sources, light cleaning
    __sources.yml
    stg_<source>__<table>.sql
  intermediate/  # business-specific reusable logic
    int_<concept>__<verb>.sql
  marts/         # narrow, consumer-facing
    core/        # dimensional / source-of-truth
    finance/     # domain-specific
    marketing/
  utils/         # macros only
```

| Layer | Purpose | Materialization | Naming |
|---|---|---|---|
| Staging | One model per source table; rename columns, cast types, no joins | view | `stg_<source>__<table>` |
| Intermediate | Reusable joins/aggregations | ephemeral or view | `int_<topic>__<verb>` (e.g. `int_orders__joined`) |
| Marts | Final tables consumers query | table or incremental | `dim_<entity>`, `fct_<event>`, or domain table |

**Rule:** stage everything from sources. No model outside `staging/` references `source()`; they reference `ref()` to a staging model. This makes source changes a single-file edit.

## sources vs refs

```yaml
# models/staging/__sources.yml
version: 2
sources:
  - name: stripe
    schema: raw_stripe
    tables:
      - name: customers
      - name: subscriptions
      - name: invoices
        loaded_at_field: created_at
        freshness:
          warn_after:  { count: 24, period: hour }
          error_after: { count: 48, period: hour }
```

```sql
-- models/staging/stripe/stg_stripe__customers.sql
with raw as (
  select * from {{ source('stripe', 'customers') }}
)
select
  id            as customer_id,
  email,
  created       as created_at,
  metadata
from raw
```

Then everything downstream references `{{ ref('stg_stripe__customers') }}`.

## Tests (the part nobody writes)

Every staging model gets at least: `unique` + `not_null` on the primary key, `accepted_values` on enums, `relationships` on foreign keys.

```yaml
# models/staging/stripe/_models.yml
version: 2
models:
  - name: stg_stripe__customers
    columns:
      - name: customer_id
        tests: [unique, not_null]
      - name: email
        tests: [not_null]
  - name: stg_stripe__subscriptions
    columns:
      - name: subscription_id
        tests: [unique, not_null]
      - name: status
        tests:
          - accepted_values:
              values: [trialing, active, past_due, canceled, unpaid]
      - name: customer_id
        tests:
          - relationships:
              to: ref('stg_stripe__customers')
              field: customer_id
```

Run `dbt test` in CI on every PR. A failing test should block merge.

## Incremental models (when the table is huge)

```sql
-- models/marts/fct_events.sql
{{
  config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns'
  )
}}

select
  event_id,
  user_id,
  event_type,
  occurred_at
from {{ ref('stg_events__raw') }}
{% if is_incremental() %}
  where occurred_at > (select coalesce(max(occurred_at), '1970-01-01'::timestamp) from {{ this }})
{% endif %}
```

Default to `merge` strategy (idempotent); fall back to `append` for true append-only logs. `unique_key` is mandatory or you'll get duplicates on re-run.

## Macros (small, one purpose each)

```sql
-- macros/dollars_to_paise.sql
{% macro dollars_to_paise(col) %}
  cast({{ col }} * 100 as integer)
{% endmacro %}
```

Used as `{{ dollars_to_paise('amount') }}`. Macros longer than 30 lines are usually doing too much.

## Snapshots (for slowly-changing dimensions)

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}
{{
  config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at'
  )
}}
select * from {{ source('stripe', 'customers') }}
{% endsnapshot %}
```

Capture the history of customer changes (plan changes, email updates). Run nightly. Build SCD Type 2 dims off these.

## Project config

```yaml
# dbt_project.yml
name: acme_warehouse
version: 1.0.0
profile: acme

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

models:
  acme_warehouse:
    staging:      { +materialized: view, +schema: staging }
    intermediate: { +materialized: ephemeral }
    marts:
      core:       { +materialized: table, +schema: core }
      finance:    { +materialized: table, +schema: finance }
```

`+schema:` lets you separate output schemas per layer — analysts can grant access to `marts.core` only.

## CI for dbt

```yaml
# .github/workflows/dbt-ci.yml
on: [pull_request]
jobs:
  dbt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install dbt-snowflake==1.8.*    # pin
      - run: dbt deps
      - run: dbt build --select state:modified+ --defer --state ./target-prod
        env: { DBT_PROFILES_DIR: . }
```

`state:modified+ --defer` only runs models you changed (and downstream). Saves CI time and warehouse credits.

## Anti-patterns

- **No staging layer; models reference `source()` everywhere** — one upstream rename is a global refactor.
- **No tests** — at minimum `unique`+`not_null` on every PK. Tests are the contract.
- **Marts that join other marts** — circular-feeling lineage. Marts should depend on intermediate or staging only.
- **Incremental without `unique_key`** — duplicate rows on re-run.
- **Snapshots run as part of `dbt run`** — they should be separate (`dbt snapshot`) on a different schedule.
- **All models materialized as table** — slow CI, expensive. Use views and ephemeral aggressively in upper layers.
- **Mixed naming conventions** — `Customer`, `customers`, `dim_customer`, `cust_dim`. Pick one and enforce in PR review.
- **No `--select state:modified+ --defer` in CI** — full builds on every PR are slow and expensive.
- **Source freshness not configured** — silent staleness. Fail loudly when upstream is broken.
- **Macros that mask SQL too much** — junior analysts can't read `{{ flexible_cohort_filter(...) }}` macros that hide 100 lines. Prefer plain SQL.

## Verify it worked

- [ ] `dbt build` runs cleanly on a fresh clone.
- [ ] `dbt test` reports zero failures and at least one test per staging model.
- [ ] No model in `models/intermediate` or `models/marts` calls `source()` directly.
- [ ] Marts schema has only consumer-ready tables; staging schema is hidden from analysts.
- [ ] CI runs `state:modified+ --defer` against prod manifest; PRs only rebuild what changed.
- [ ] Source freshness check fires alerts when upstream lags > 2x SLA.
- [ ] Incremental models use `unique_key`; re-running them is idempotent (no dup rows).
- [ ] At least one snapshot captures SCD-type-2 history of a key dim.
- [ ] `dbt docs generate && dbt docs serve` produces a navigable lineage graph.
- [ ] Layered schemas (`staging.`, `marts.`) — analysts only have access to marts.
