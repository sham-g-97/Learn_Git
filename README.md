# dbt Project Gap Analysis & Production Readiness Guide

## Current State Summary

| Aspect | Status | Detail |
|--------|--------|--------|
| **Models** | 204 total | staging (65), integrations (47), data_marts (37), data_warehouse (21), hubspot_integrations (18), data_marts_hubspot (10), interim (6) |
| **Schema Layering** | Good | 7 schemas, clean directory structure, custom `generate_schema_name` macro |
| **Tags** | Good | ~80% of models tagged for selective execution |
| **Incremental Models** | Good | 23+ staging models use incremental with `modified` timestamp CDC |
| **Indexes** | Good | 63 models define btree indexes on join columns |
| **Unique Keys** | Partial | 53 of 204 models define `unique_key` |
| **Source Definitions** | Good | 13 source yml files covering 9+ FDW databases |
| **Seeds** | Good | 3 CSV seeds with column typing in properties.yml |
| **Macros** | Good | 8 custom macros (schema naming, PK constraints, date utilities) |
| **Packages** | Minimal | Only `dbt_date` (0.10.1) and `dbt_utils` (1.1.1) |

---

## What's Missing

### 1. Testing — CRITICAL (Zero Tests Exist)

**Current state:** The `tests/` directory is empty. No generic tests, no singular tests, no unit tests. The pipeline runs `dbt seed && dbt run` — never `dbt test` or `dbt build`. Even if tests were added today, they would never execute.

**Why it matters:** Without tests, there is no automated way to detect broken assumptions — duplicate primary keys, null foreign keys, unexpected enum values, or broken referential integrity. Data issues propagate silently to Metabase dashboards and downstream integrations (HubSpot, Planhat).

**What's needed:**

#### 1.1 Primary Key Tests (Highest Priority)

Every model with a `unique_key` config needs matching `unique` + `not_null` tests in a yml file. These 53 models already declare their PK — the tests just aren't defined.

```yaml
# Example: models/data_warehouse/_data_warehouse__models.yml
version: 2

models:
  - name: fact_events
    columns:
      - name: auction_id
        tests:
          - unique
          - not_null

  - name: dim_awards
    columns:
      - name: auction_id
        tests:
          - unique
          - not_null

  - name: dim_purchaser_organisations
    columns:
      - name: organisation_id
        tests:
          - unique
          - not_null
```

**Scope:** Create one `_<directory>__models.yml` file per model subdirectory (approx 30 yml files across all layers).

#### 1.2 Relationship / Referential Integrity Tests

The data_warehouse layer is a star schema. Every foreign key should have a `relationships` test to ensure joins won't silently drop rows.

Key relationships to test:

| Model | FK Column | References |
|-------|-----------|------------|
| fact_events | organisation_id | dim_purchaser_organisations.organisation_id |
| dim_awards | auction_id | fact_events.auction_id |
| dim_bidder_organisations | auction_id | fact_events.auction_id |
| dim_scenario_solutions | auction_id | fact_events.auction_id |
| dim_spend | auction_id | fact_events.auction_id |
| dim_spend_metrics | auction_id | fact_events.auction_id |
| All data_marts | event_id / auction_id | fact_events.auction_id |

```yaml
# Example relationship test
columns:
  - name: organisation_id
    tests:
      - not_null
      - relationships:
          to: ref('dim_purchaser_organisations')
          field: organisation_id
```

#### 1.3 Accepted Values Tests

Categorical columns with known enum values should have `accepted_values` tests. Examples:

```yaml
columns:
  - name: auction_state
    tests:
      - accepted_values:
          values: ['DRAFT', 'OPEN', 'PAUSED', 'CLOSED', 'ARCHIVED']
          config:
            severity: warn

  - name: auction_type
    tests:
      - accepted_values:
          values: ['RFQ', 'ENGLISH_AUCTION', 'DUTCH_AUCTION', 'JAPANESE_AUCTION', 'RFI']
          config:
            severity: warn
```

#### 1.4 Source Tests

All 13 source yml files list table names but have zero column-level tests. At minimum, add `unique` + `not_null` on the primary key of every source table.

#### 1.5 Test Configuration (dbt_project.yml)

```yaml
tests:
  bi_warehouse:
    +severity: warn           # Start with warn, promote to error after validation period
    +store_failures: true     # Write failures to DB for monitoring
    +schema: test_results     # Dedicated schema for failure records
```

#### 1.6 Singular / Custom Tests

Create custom SQL tests in `tests/` for business rules that generic tests can't express:

```sql
-- tests/assert_no_duplicate_events_in_report.sql
-- Events report should have exactly one row per event
SELECT event_id, COUNT(*) as cnt
FROM {{ ref('events_report') }}
GROUP BY event_id
HAVING COUNT(*) > 1
```

#### 1.7 Unit Tests (dbt 1.8+)

For complex transformation logic (e.g., `int_bid_automation_rate.sql`, `CAT_custom_report.sql`), define unit tests with mocked inputs:

```yaml
unit_tests:
  - name: test_bid_automation_rate_calculation
    model: int_bid_automation_rate
    given:
      - input: ref('stg_bid_automation_rate')
        rows:
          - {rate_card_id: 1, rate: 100.0, currency: 'USD'}
    expect:
      rows:
        - {rate_card_id: 1, rate_usd: 100.0}
```

---

### 2. Documentation — CRITICAL (Zero Documentation Exists)

**Current state:** No column descriptions on any model. No `.md` doc blocks. Only 2 yml files (`dim_awards.yml`, `fact_events.yml`) define column data types — but no descriptions. Running `dbt docs generate` would produce a catalog with column names and types only, no business context.

**Why it matters:** Without documentation, new team members, analysts, and (critically) LLMs have no way to understand what columns mean, what the grain of a table is, or how tables relate to each other.

**What's needed:**

#### 2.1 Model-Level Descriptions

Every model needs a `description` field in its yml entry explaining:
- What the table represents
- What the grain is (one row per what?)
- Key business context

```yaml
models:
  - name: fact_events
    description: >
      Central fact table for all sourcing events (auctions). One row per auction_id.
      Contains event lifecycle states, timing, organization ownership, automation
      classification, bidder statistics, and template information. This is the primary
      join point for the star schema — all dimension tables reference auction_id.

  - name: events_report
    description: >
      Pre-denormalized report table for the Metabase events dashboard. One row per
      sourcing event with organization details, bidder counts, lot counts, scenario
      evaluations, and spend metrics already joined in. Filter on
      is_bi_filtered_test = false for production data.
```

#### 2.2 Column-Level Descriptions

At minimum, document:
- All primary keys
- All foreign keys
- All boolean flags (explain what true/false means)
- All categorical columns (list valid values)
- All calculated/derived columns (explain the formula)
- All currency/financial columns (explain currency, normalization)

```yaml
columns:
  - name: auction_id
    description: "Unique identifier for each sourcing event. Originates from procurement service design_auction.id."

  - name: is_bi_filtered_test
    description: "True = event is a test event excluded from production BI reports. More reliable than is_test as it includes additional Keelvar-internal org filtering."

  - name: event_spend_in_usd
    description: "Total event spend normalized to USD using yearly exchange rates from seed_currency_exchange_rate. Null if no spend data exists."

  - name: bidder_participation_rate
    description: "Ratio of orgs_that_placed_bid / bidder_orgs_connected. Ranges 0.0-1.0. Null if no bidders connected."
```

#### 2.3 Doc Blocks (Reusable Descriptions)

Create `models/docs.md` for descriptions used across multiple models:

```markdown
{% docs auction_id %}
Unique integer identifier for a sourcing event (auction). This is the primary grain
of the fact_events table and the main join key across the star schema. Originates from
the procurement service's `design_auction.id` column.
{% enddocs %}

{% docs organisation_id %}
Integer identifier for a purchaser organization. Foreign key to
dim_purchaser_organisations. Represents the company that owns/created the sourcing event.
{% enddocs %}

{% docs is_bi_filtered_test %}
Boolean flag indicating whether BI reporting considers this event a test.
When true, the event is excluded from production Metabase reports.
More reliable than `is_test` for BI purposes as it includes additional filtering logic.
{% enddocs %}
```

Reference in yml: `description: '{{ doc("auction_id") }}'`

#### 2.4 Source Descriptions

All 13 source files list tables with no descriptions. Add descriptions for each source and its tables:

```yaml
sources:
  - name: procurement
    description: >
      Foreign Data Wrapper tables from the procurement service database (raw_proc schema).
      Contains core sourcing event data: auctions, organisations, lots, bids, scenarios,
      and awards. Updated in real-time via FDW.
    tables:
      - name: foreign_design_auction
        description: "Core auction/event design table. One row per sourcing event created in the procurement service."
      - name: foreign_organisation_organisation
        description: "Organization master data. One row per org registered in the procurement platform."
```

#### 2.5 Exposures (Downstream Consumers)

Create `models/exposures.yml` to document what consumes your dbt models:

```yaml
version: 2

exposures:
  - name: events_dashboard
    type: dashboard
    maturity: high
    url: "<metabase-url>/dashboard/events"
    description: "Main events overview dashboard in Metabase"
    depends_on:
      - ref('events_report')
      - ref('dim_date')
    owner:
      name: BI Team
      email: bi-team@keelvar.com

  - name: hubspot_sync
    type: application
    maturity: high
    description: "HubSpot CRM sync pipeline (Lambda)"
    depends_on:
      - ref('hubspot_companies')
      - ref('hubspot_contacts')
      - ref('hubspot_deals')
    owner:
      name: BI Team
      email: bi-team@keelvar.com

  - name: planhat_nps_sync
    type: application
    description: "Planhat NPS score sync (Lambda)"
    depends_on:
      - ref('purchaser_nps')
      - ref('bidder_nps')
    owner:
      name: BI Team
      email: bi-team@keelvar.com

  - name: llm_event_categorization
    type: application
    description: "Bedrock LLM service for event categorization"
    depends_on:
      - ref('as_events_for_llm_tagging')
      - ref('so_events_for_llm_event_tagging')
    owner:
      name: BI Team
      email: bi-team@keelvar.com
```

**Why exposures matter:** They show up in the DAG, making it clear which models are "public API" and cannot be changed without impact analysis. LLMs can also use exposures to understand what each table powers.

#### 2.6 Tooling: dbt-osmosis

Use `dbt-osmosis` to auto-scaffold yml files for all 204 models in a single pass:

```bash
pip install dbt-osmosis
dbt-osmosis yaml refactor --profiles-dir ... --project-dir ...
```

This pulls column names and data types from the database catalog and generates yml stubs. Then enrich manually with descriptions and meta blocks. **Saves 2-3 days of manual work.**

---

### 3. LLM Metadata Readiness — HIGH PRIORITY (Zero Meta Blocks Exist)

**Current state:** No `meta:` blocks anywhere in the project. The `manifest.json` (already uploaded to S3 at `bi-reporting-dbt-manifests`) contains model/column names and types, but zero business context.

**Why it matters:** When you hook an LLM on top of the warehouse, it needs to understand: What does each table represent? What's the grain? Which columns are PKs, FKs, measures, dimensions? What are valid enum values? What are common filters? All of this should live in dbt metadata so the manifest.json becomes a self-describing semantic layer.

**What's needed:**

#### 3.1 Standard Model-Level Meta Schema

Define and apply consistently across data_warehouse and data_marts:

```yaml
models:
  - name: fact_events
    meta:
      # Structural metadata
      layer: data_warehouse
      grain: "one row per sourcing event (auction_id)"
      primary_key: auction_id
      semantic_type: fact_table  # fact_table | dimension | bridge | staging | report

      # Business metadata
      business_domain: sourcing_events  # sourcing_events | organizations | bidding | awards | rate_management | bots | hubspot | spend
      business_owner: "BI Team"
      update_frequency: daily
      contains_pii: false
      data_classification: internal  # internal | confidential | public

      # LLM-specific metadata
      llm_summary: >
        Central fact table for all sourcing events. Each row is one auction.
        Join to dim_purchaser_organisations on organisation_id for org details.
        Join to dim_awards on auction_id for award/savings data.
        Filter is_bi_filtered_test = false for production data.
      common_filters:
        - "is_bi_filtered_test = false  -- exclude test events"
        - "auction_state = 'CLOSED'  -- only completed events"
      common_joins:
        - "dim_purchaser_organisations ON organisation_id"
        - "dim_awards ON auction_id"
        - "dim_date ON created_date_key"
      key_metrics:
        - "bidder_participation = orgs_that_placed_bid / bidder_orgs_connected"
        - "event_count = COUNT(DISTINCT auction_id)"
      business_rules:
        - "is_bi_filtered_test=true events are internal test events"
        - "event_spend_in_usd uses yearly exchange rates from seed_currency_exchange_rate"

      # Consumer metadata
      downstream_consumers: [metabase, hubspot_sync, planhat_sync, llm_service]
```

#### 3.2 Standard Column-Level Meta Schema

```yaml
columns:
  - name: auction_id
    description: "Unique identifier for each sourcing event"
    meta:
      semantic_type: primary_key  # primary_key | foreign_key | measure | dimension | categorical | boolean_flag | timestamp | text | currency_amount | percentage | count | identifier
      pii: false
      nullable: false
      grain_role: primary_key  # primary_key | degenerate_dimension | measure | attribute

  - name: organisation_id
    description: "FK to dim_purchaser_organisations"
    meta:
      semantic_type: foreign_key
      references: "dim_purchaser_organisations.organisation_id"
      pii: false

  - name: auction_state
    description: "Current lifecycle state of the event"
    meta:
      semantic_type: categorical
      enum_values: ['DRAFT', 'OPEN', 'PAUSED', 'CLOSED', 'ARCHIVED']
      example_values: ['OPEN', 'CLOSED']

  - name: event_spend_in_usd
    description: "Total spend normalized to USD"
    meta:
      semantic_type: currency_amount
      unit: USD
      nullable: true
      aggregation: sum
```

#### 3.3 Priority Order for Metadata

LLMs will primarily query the **data_warehouse** and **data_marts** layers. Document these first:

**Tier 1 — Document immediately (the "LLM-facing" tables):**
- `fact_events` (central fact, ~80 columns)
- `dim_purchaser_organisations` (org dimension)
- `dim_awards` (awards/savings)
- `dim_spend` / `dim_spend_metrics` (financials)
- `events_report` (main Metabase source)
- `agents_report` / `agents_requests_report` (bot analytics)
- `bid_automation_report` / `bid_automation_execution_report`

**Tier 2 — Remaining data_warehouse + data_marts:**
- All 21 data_warehouse models
- All 37 data_mart models

**Tier 3 — Staging and integrations:**
- Use dbt-osmosis to scaffold, enrich over time

#### 3.4 LLM Context Document Generator

Create a script that parses `manifest.json` and produces a condensed context document for LLM consumption (the full manifest can be 10+ MB):

```
TABLE: data_marts.events_report
GRAIN: one row per sourcing event
PRIMARY KEY: event_id
DESCRIPTION: Main denormalized report table...
COLUMNS:
  - event_id (bigint, PK): Unique identifier for each sourcing event
  - organisation_name (text): Name of the purchaser organization
  - event_state (text, enum: DRAFT|OPEN|PAUSED|CLOSED|ARCHIVED): Current lifecycle state
  - event_spend_in_usd (numeric, currency:USD): Total spend normalized to USD
RELATIONSHIPS:
  - organisation_id -> dim_purchaser_organisations.organisation_id
COMMON FILTERS: is_bi_filtered_test = false
```

#### 3.5 Future: dbt Semantic Layer (MetricFlow)

For a later phase, consider adopting dbt's Semantic Layer for standardized metric definitions that LLMs can query programmatically:

```yaml
semantic_models:
  - name: events
    model: ref('fact_events')
    entities:
      - name: event
        type: primary
        expr: auction_id
      - name: organisation
        type: foreign
        expr: organisation_id
    dimensions:
      - name: auction_state
        type: categorical
      - name: created
        type: time
        type_params:
          time_granularity: day
    measures:
      - name: event_count
        agg: count_distinct
        expr: auction_id

metrics:
  - name: total_events
    type: simple
    type_params:
      measure: event_count
  - name: bidder_participation_rate
    type: derived
    type_params:
      expr: orgs_that_placed_bid / NULLIF(bidder_orgs_connected, 0)
```

---

### 4. Source Freshness — HIGH PRIORITY (Not Configured)

**Current state:** 13 source yml files define table names but no freshness checks. No way to detect if a Foreign Data Wrapper connection breaks.

**What's needed:**

Add `loaded_at_field` and freshness thresholds to every source:

```yaml
sources:
  - name: procurement
    schema: raw_proc
    description: "FDW tables from procurement service"
    freshness:
      warn_after: {count: 6, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: modified
    tables:
      - name: foreign_design_auction
        loaded_at_field: modified
      - name: foreign_organisation_organisation
        loaded_at_field: modified
      # Tables without 'modified' column:
      - name: foreign_design_tag
        loaded_at_field: created
```

Add `dbt source freshness` as a pre-check in the pipeline:

```bash
dbt source freshness --profiles-dir ... --project-dir ... || echo "Freshness warnings detected"
dbt build --profiles-dir ... --project-dir ...
```

---

### 5. Data Contracts — MEDIUM PRIORITY (Not Enforced)

**Current state:** `dim_awards.yml` and `fact_events.yml` define column data types with `contract: enforced: false`. The other 202 models have no contracts at all.

**What's needed:**

#### 5.1 Enforce Contracts on data_warehouse Layer

The 21 data_warehouse models are the "public API" of the warehouse. They should have enforced contracts preventing accidental column changes:

```yaml
models:
  - name: fact_events
    config:
      contract:
        enforced: true
    columns:
      - name: auction_id
        data_type: bigint
      - name: organisation_id
        data_type: bigint
      - name: auction_state
        data_type: text
      # ... all columns must be listed with data_type
```

#### 5.2 Do NOT Enforce on Staging/Integrations

Staging and integration models change frequently as source schemas evolve. Contracts here would create unnecessary friction.

#### 5.3 Optionally Enforce on Data Marts

Data marts that power Metabase dashboards could benefit from contracts to prevent accidental column drops that break dashboards.

---

### 6. Production Pipeline — HIGH PRIORITY

**Current state:**
- Pipeline runs `dbt seed && dbt run` — never executes tests
- `threads: 1` in profiles.yml — fully serializes 204 models
- No source freshness pre-check
- No manifest upload after build

**What's needed:**

#### 6.1 Switch to `dbt build`

`dbt build` runs seeds, models, snapshots, and tests in DAG order. This is the modern standard.

```bash
# Before (current)
dbt seed --profiles-dir ... --project-dir ...
dbt run --profiles-dir ... --project-dir ...

# After
dbt build --profiles-dir ... --project-dir ...
```

#### 6.2 Increase Thread Count

```yaml
# profiles.yml
bi_warehouse:
  target: dev
  outputs:
    dev:
      threads: 4  # Was 1. Test with 4, increase to 8 if DB handles it.
```

The DAG has natural parallelism — 11 staging source domains and 13 integration subdirectories can run concurrently.

#### 6.3 Full Pipeline Script

```bash
#!/bin/bash
set -e
DBT_ARGS="--profiles-dir ... --project-dir ..."

# 1. Pre-check source freshness
dbt source freshness $DBT_ARGS || echo "Freshness warnings detected"

# 2. Build everything (seed + run + test + snapshot)
dbt build $DBT_ARGS

# 3. Generate and upload docs/manifest
dbt docs generate $DBT_ARGS
# Upload manifest.json to S3 for LLM consumption
```

---

### 7. Governance — MEDIUM PRIORITY

**Current state:**
- No `group` or `access` configurations
- Post-hook GRANTs duplicated across 37+ SQL model files
- No ownership definitions

**What's needed:**

#### 7.1 Groups and Access (dbt 1.5+)

```yaml
# dbt_project.yml
groups:
  - name: core_warehouse
    owner:
      name: BI Team
      email: bi-team@keelvar.com
  - name: hubspot_pipeline
    owner:
      name: BI Team
      email: bi-team@keelvar.com
```

```yaml
# In model yml
models:
  - name: fact_events
    config:
      group: core_warehouse
    access: public       # Can be ref'd by any group

  - name: int_events
    config:
      group: core_warehouse
    access: protected    # Only within core_warehouse

  - name: stg_procurement_auctions
    config:
      group: core_warehouse
    access: private      # Only within same directory
```

**Access rules:**
- `data_warehouse` + `data_marts` → `public` (the API)
- `integrations` → `protected`
- `staging` → `private`

#### 7.2 Consolidate Post-Hooks

Move scattered `GRANT SELECT` post-hooks from individual SQL files to `dbt_project.yml`:

```yaml
models:
  bi_warehouse:
    data_marts:
      +post-hook:
        - "GRANT SELECT ON {{ this }} TO metabase_prod_user"
    data_marts_hubspot:
      +post-hook:
        - "GRANT SELECT ON {{ this }} TO metabase_prod_user"
```

Then remove the individual `post_hook` configs from each SQL model.

---

### 8. Snapshots — LOW PRIORITY

**Current state:** `snapshots/` directory exists but is empty. SCD2 (Slowly Changing Dimensions Type 2) is currently implemented in Python (`spend_analytics/`).

**What's needed (optional):**

For simpler SCD2 use cases, dbt snapshots are easier to maintain:

```sql
-- snapshots/snap_organisations.sql
{% snapshot snap_organisations %}
{{
    config(
      target_schema='snapshots',
      unique_key='organisation_id',
      strategy='timestamp',
      updated_at='modified',
    )
}}
SELECT * FROM {{ ref('dim_purchaser_organisations') }}
{% endsnapshot %}
```

This is lower priority since the Python pipeline works. Consider migrating when starting the new project.

---

### 9. Materialization Optimization — LOW PRIORITY

**Current state:** All models materialized as `table` in `dbt_project.yml`, though ~23 staging models override to `incremental` in their SQL config.

**What's needed:**

~32 non-incremental staging models read directly from FDW sources. As `view` materialization, they:
- Always show the latest FDW data (no stale tables)
- Use zero storage
- Execute faster during `dbt build` (no table creation)

```yaml
# dbt_project.yml
models:
  bi_warehouse:
    staging:
      +materialized: view  # Change from table
```

Models that need `table` or `incremental` already override in their SQL config block.

---

### 10. CI/CD for dbt — LOW PRIORITY

**Current state:** CI runs `sqlfluff lint` on PRs but no dbt-specific checks.

**What's needed:**

Add a CI job for PRs touching `src/warehouse_ecs/dbt/`:

```yaml
# GitHub Actions example
- name: dbt parse
  run: dbt parse --profiles-dir ... --project-dir ...  # Validates syntax

- name: dbt compile
  run: dbt compile --profiles-dir ... --project-dir ...  # Ensures refs/sources resolve

- name: sqlfluff lint
  run: poetry run sqlfluff lint .  # Already exists

# Optional: state-based testing against test DB
- name: dbt test (modified only)
  run: dbt test --select state:modified+ --defer --state ./prod-manifest/ ...
```

---

### 11. Miscellaneous Issues

#### 11.1 Stray Files
Two JSON files appear to be accidental commits:
- `models/integrations/awards/sss.json`
- `models/integrations/awards/ssss.json`

These should be removed.

#### 11.2 dbt Packages to Consider

| Package | Purpose |
|---------|---------|
| `dbt-labs/dbt_utils` | Already installed. Keep. |
| `calogica/dbt_date` | Already installed. Keep. |
| `dbt-labs/dbt_expectations` | Great Expectations-style tests (distribution checks, regex validation, row count comparisons) |
| `elementary-data/elementary` | Data observability — auto-generates anomaly detection tests, volume monitoring, freshness alerts |
| `dbt-labs/codegen` | Auto-generates yml scaffolding, base models, source definitions |

---

## Implementation Priority Matrix

| Phase | What | Priority | Effort | Impact |
|-------|------|----------|--------|--------|
| 0 | Fix pipeline (`dbt build`), increase threads, remove stray files | **P0** | 2 hours | Immediate perf + correctness |
| 1 | Primary key + relationship tests on data_warehouse + data_marts | **P0** | 3-4 days | Data trust |
| 2 | Model + column descriptions on data_warehouse + data_marts | **P1** | 4-5 days | Team productivity + LLM readiness |
| 3 | Meta blocks on data_warehouse + data_marts (LLM metadata) | **P1** | 3-4 days | LLM integration |
| 4 | Source freshness configuration | **P1** | 1 day | Production hardening |
| 5 | Exposures for Metabase/integrations | **P1** | 1 day | Impact analysis |
| 6 | Data contracts on data_warehouse layer | **P2** | 2 days | Schema stability |
| 7 | Governance (groups, access, consolidated hooks) | **P2** | 2 days | Maintainability |
| 8 | Remaining documentation (staging + integrations via dbt-osmosis) | **P2** | 3 days | Completeness |
| 9 | CI/CD dbt checks | **P2** | 1-2 days | Regression prevention |
| 10 | Snapshots, materialization optimization | **P3** | 2 days | Nice to have |

**Total estimated effort: ~4-5 weeks for one engineer.**

**Acceleration tip:** Use `dbt-osmosis yaml refactor` to auto-scaffold all yml files in one pass (saves 2-3 days), then enrich manually starting with data_warehouse and data_marts.

---

## Checklist for Every New Model (Going Forward)

- [ ] yml entry with `description` and `meta` (layer, grain, primary_key, semantic_type, llm_summary)
- [ ] Column descriptions for at minimum: primary key, all foreign keys, all boolean flags, all calculated fields
- [ ] `unique` + `not_null` tests on primary key
- [ ] `relationships` tests for all foreign keys
- [ ] `accepted_values` tests for categorical columns with known enums
- [ ] Follows naming convention: `stg_`, `int_`, `dim_`, `fact_`, or descriptive report name
- [ ] Column-level `meta` with `semantic_type`, `pii`, `references` (for FKs)
- [ ] If consumed by Metabase: exposure entry in `exposures.yml`
- [ ] If data_warehouse layer: enforced data contract with data types
