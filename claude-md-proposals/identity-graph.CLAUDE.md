# CLAUDE.md — resonate/identity-graph
# This is a NEW file to be created at the root of the identity-graph repo

# identity-graph

## Project Purpose and Architecture Overview

This repo contains the **Identity Graph** — Resonate's Person Resolution and Identity System (PRISM). It has two layers:

1. **Scala/Spark Jobs** (`src/main/scala/`) — 11 batch ETL jobs that build the underlying identity tables (identifier index, persons) from raw sources (L2, Experian, TapAd, LiveIntent, etc.)
2. **prism_dbt** (`dbt/prism_dbt/`) — PRISM's consumer-facing dbt package v1.0. Exposes four primitives (`waterfall_match`, `identifier_expand`, `persons_project`, `lookup`) as Lambda-invokable Snowflake service models.

**Production JAR path:** `s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar`

**Design docs (published to S3):**
- `https://s3.nonprod.aws.resonatedigital.net/person-identity-graph/index.html` — hub
- `docs/design/design_and_production_readiness.html` — main architecture doc
- `docs/design/consumer_interface_design.html` — consumer integration (Append/BlockGraph/GeoFix/Cortex)
- `docs/design/project_plan.html` — 6 tracks, Jira ticket DAG

---

## Repository Structure

```
src/main/scala/com/resonate/spark/apps/
  ExperianPreprocessJob.scala         # Normalizes Experian ConsumerView + digital graph
  L2ConsumerPreprocessJob.scala       # Preprocesses L2 consumer tab-delimited files
  TapadIndexJob.scala                 # Experian/TapAd → identifier_index
  L2IndexJob.scala                    # L2 voter → identifier_index
  L2ConsumerIndexJob.scala            # L2 consumer → identifier_index
  OnboardingMatchJob.scala            # 4-phase matching (blocking, corroboration, scoring, fallback)
  OnboardingHouseholdLinksJob.scala   # Household CTV/TTD attribution via digital graph
  OnboardingPersonsJob.scala          # TapAd-only person records (deprecated: BL-106)
  MergeOnboardingLinksJob.scala       # Multi-provider collapse + confidence scoring
  PersonIdentityJob.scala             # Main orchestrator: persons + identifier_index
  CustomerMatchJob.scala              # Ephemeral customer file matching (7 fallback tiers)

src/main/scala/com/resonate/spark/utils/
  HashUtils.scala                     # Deterministic SHA-256 person_id + household_id
  StagingWriter.scala                 # Cache/count/write/unpersist pattern
  AddressNormalizer.scala             # USPS Pub 28 normalization + nickname resolution (max 5 hops)
  IpFilter.scala                      # RFC1918 + cloud IP blocklist
  ScoringConfig.scala                 # Centralized confidence scoring

dbt/prism_dbt/                        # Consumer-facing dbt package (see dbt/prism_dbt/README.md)
  macros/                             # waterfall_match, identifier_expand, persons_project, lookup
  models/service/                     # Lambda-invokable: waterfall.sql, identifier_expand.sql, persons_project.sql
  snowflake/
    function/name_address_hash/       # NAME_ADDRESS_HASH Snowflake UDF
    stored_procedure/prism_lookup/    # PRISM_LOOKUP SP
    stored_procedure/name_address_lookup/ # NAME_ADDRESS_LOOKUP SP

docs/design/                          # HTML design artifacts (published to S3)
docs/scripts/                         # build_columns_page.py, sample_parquet.py

.github/workflows/
  snowflake_dbt.yml                   # Deploy prism_dbt via snow dbt deploy
  snowflake_stored_procedure.yml      # Deploy PRISM_LOOKUP / NAME_ADDRESS_LOOKUP
  snowflake_function.yml              # Deploy NAME_ADDRESS_HASH UDF
```

---

## Key Commands

### Scala/Spark Jobs (SBT)

```bash
# Run all unit tests (443 tests across 20 suites)
sbt test

# Run a specific test suite
sbt "testOnly com.resonate.spark.apps.PersonIdentityJobSpec"
sbt "testOnly com.resonate.spark.apps.CustomerMatchJobSpec"

# Build assembly fat JAR for EMR deployment
sbt assembly

# Upload to S3 (prod)
aws s3 cp target/identity-graph-latest.jar \
  s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar
```

See the [deployment wiki](https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/3683057840/Pipeline+Scala+SBT+GitOps) for the GitOps deploy process.

### prism_dbt (Snowflake Native dbt Package)

```bash
cd dbt/prism_dbt

# Install dbt dependencies
dbt deps

# Local dev (requires ~/.dbt/profiles.yml — copy from profiles.yml.example)
dbt seed --profiles-dir <your-profiles-dir>
dbt run  --select waterfall --profiles-dir <your-profiles-dir>
dbt test --profiles-dir <your-profiles-dir>

# Offline unit tests only (no Snowflake needed — fast)
dbt test --select test_type:unit      # 16 tests

# All tests (includes live Snowflake singular tests against resonate-dev)
dbt test                              # 25 tests total
```

### Snowflake Deployments (GitHub Actions)

All Snowflake deployments go through GitHub Actions. Do not run `snow dbt deploy` manually from the CLI.

| Workflow | What it deploys |
|---|---|
| `snowflake_dbt.yml` | `prism_dbt` package via `snow dbt deploy` (runs `dbt parse` as fast-fail gate) |
| `snowflake_stored_procedure.yml` | `PRISM_LOOKUP` or `NAME_ADDRESS_LOOKUP` stored procedures |
| `snowflake_function.yml` | `NAME_ADDRESS_HASH` UDF |

---

## PRISM dbt Package (prism_dbt v1.0)

### The Four Primitives

| Macro | Direction | Purpose |
|---|---|---|
| `waterfall_match` | identifiers → 1 person_id per row | Resolve mixed-identifier customer rows to a canonical `person_id`. First-match-wins by priority. |
| `identifier_expand` | person_id → identifiers of ONE type | Fan out person_ids to deliverable identifiers. Call once per identifier type. |
| `persons_project` | person_id → persons attributes | Project selected `persons` columns (name, address, lalvoterid, …) onto person_ids. |
| `lookup` | one identifier → one person + linked IDs | Single-row debug. Also deployed as `PRISM_LOOKUP` stored procedure. |

### v1.0 Data Sources
- `PERSONS_LATEST` + `IDENTIFIER_INDEX_LATEST` (filtered to `identifier_rank = 1`)
- `CROSSWALK_LATEST` exists but is **NOT used** in v1.0 (missing metadata for confidence filtering)
- `ZIP11` routes automatically to `persons.zip11` (not `identifier_index`) via `_internal/lookup_strategy.sql`

### Service Model Invocation

```sql
EXECUTE DBT PROJECT RESONATE.PRISM.prism_dbt
  ARGS = $$run --select waterfall --vars '{
    "input_relation":  "MY_DB.MY_SCHEMA.my_input",
    "consumer":        "append",
    "job_run_id":      "run_001",
    "waterfall_order": [
      {"identifier_type": "HEM_MD5",    "stitch_label": "hem"},
      {"identifier_type": "HEM_SHA256", "stitch_label": "hem"},
      {"identifier_type": "MAID",       "stitch_label": "maid"}
    ]
  }'$$;
```

### NAME_ADDRESS_HASH UDF

Normalizes raw PII to a SHA-256 hash for name+address waterfall steps:

```sql
SELECT NAME_ADDRESS_HASH(first_name, last_name, address_line, zip) AS NAME_ADDRESS FROM my_input;
```

v1.0: `LOWER + TRIM + strip non-alphanumeric + LPAD zip + pipe-concat + SHA-256`. Does not yet do canonical-first-name resolution (Bill ≠ William — planned v1.1).

---

## Key Concepts and Bug Fixes in v1.0

- **ZIP11 routing**: `waterfall_match` auto-routes `ZIP11` steps to `persons.zip11`. Just include `{"identifier_type": "ZIP11", "stitch_label": "zip11"}` in `waterfall_order`.
- **Fan-out tiebreak**: Always deterministic. Order: confidence DESC → source_count DESC → max_effective_weight DESC → person_id ASC.
- **`L2IndexJob` HOUSEHOLD_ID null guard**: `filterNulls = true` — prevents phantom household collapse.
- **`OnboardingMatchJob` Phase 3 tiebreak**: `ORDER BY person_id ASC` as final tiebreak — fixes non-determinism.
- **`ExperianPreprocessJob` aggregation**: `min(id_value)` not `first(id_value)` — deterministic.
- **Null guards**: `identifier_value` filtered before `groupBy` in `PersonIdentityJob` and `MergeOnboardingLinksJob`.

---

## Project-Specific Rules

- **All Snowflake deployments via GitHub Actions** — never `snow dbt deploy` from the CLI.
- **`snowflake_profiles/profiles.yml` is a deploy stub** (account/user = `'_'`). Do not put real credentials there.
- **dbt unit tests are offline** — run `--select test_type:unit` first for fast feedback.
- **Singular tests require `resonate-dev` Snowflake** — live mock tables in `RESONATE.PRISM.*` (provisioned from `.context/cdp-119018-mock-tables.sql`).
- **person_id ≠ RID**: PRISM produces `person_id`. `RID` (Resonate-internal) is owned by Append's downstream pipeline. `waterfall_match` output column is `prism_matched_person_id`.
- **SemVer**: v1.0 is the Append cutover baseline. Release tagging tracked in CDP-119015.
