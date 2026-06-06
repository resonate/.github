# identity-graph

## Project Purpose and Architecture Overview

This repo is the home of **PRISM** (Person Resolution and Identity System Map) — Resonate's person-centric identity graph. It contains two distinct sub-systems:

1. **Scala/Spark pipeline jobs** — batch EMR jobs that build the identity graph from raw L2, Experian/Tapad, LiveIntent, LiveRamp, Sovrn, and customer data.
2. **`prism_dbt` v1.0** — a Snowflake-native dbt package exposing consumer-facing macros for identity resolution (waterfall match, identifier expand, persons project, lookup).

**Top-level layout:**

```
src/
  main/scala/com/resonate/
    identity/                # Spark pipeline jobs (11 jobs)
      ExperianPreprocessJob
      L2ConsumerPreprocessJob
      TapadIndexJob
      L2IndexJob
      L2ConsumerIndexJob
      OnboardingMatchJob
      OnboardingHouseholdLinksJob
      OnboardingPersonsJob       # deprecated (BL-106)
      MergeOnboardingLinksJob
      PersonIdentityJob
      CustomerMatchJob
    shared/                  # Shared Spark utilities
      HashUtils               # SHA256-based person_id / household_id
      StagingWriter           # cache/count/write/unpersist pattern
      AddressNormalizer       # USPS Pub 28, nickname chains, soundex
      IpFilter                # RFC1918 + cloud IP blocklist
      ScoringConfig           # confidence scoring, recency decay
  test/scala/com/resonate/   # 443 unit tests across 20 test suites
dbt/
  prism_dbt/                 # dbt package (consumer-facing)
    macros/                  # 4 primitive macros + internal helpers
      waterfall_match.sql
      identifier_expand.sql
      persons_project.sql
      lookup.sql
      _internal/             # routing, validation helpers
    models/
      service/               # 3 Lambda-invokable service models
        waterfall.sql
        identifier_expand.sql
        persons_project.sql
    seeds/                   # Test seed data
    tests/                   # dbt unit_tests framework tests (33 tests)
    snowpark_udf/
      NAME_ADDRESS_HASH/     # Canonical name+address → person_id hash UDF
    snowpark_stored_procedures/
      PRISM_LOOKUP/          # Single-row debug lookup SP
      NAME_ADDRESS_LOOKUP/   # Resolve by raw PII SP
docs/
  design/                    # PRISM design + project plan HTML artifacts
    index.html               # Documentation hub
    design_and_production_readiness.html
    consumer_interface_design.html
    project_plan.html
    raw_dataset_columns.html
    id_path_explorer.html
  scripts/
    build_columns_page.py    # Regenerates raw_dataset_columns.html
.github/
  workflows/
    snowflake_dbt.yml        # Deploy prism_dbt via `snow dbt deploy`
    snowflake_stored_procedure.yml  # Deploy PRISM_LOOKUP + NAME_ADDRESS_LOOKUP SPs
    snowflake_function.yml   # Deploy NAME_ADDRESS_HASH UDF
    ci.yml                   # Run sbt tests on PRs
```

---

## Key Commands

### Scala/Spark (Identity Pipeline Jobs)

**JDK 17 is required.**

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home

sbt compile          # compile
sbt test             # 443 unit tests, 20 suites
sbt assembly         # fat jar for EMR (output: target/identity-graph.jar)
```

**Deploy JAR to S3:**
```bash
aws sso login
aws s3 cp target/identity-graph.jar \
  s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar
```

### dbt (prism_dbt Package)

```bash
# Install Snowflake CLI (snow)
# See https://docs.snowflake.com/en/developer-guide/snowflake-cli

# Validate dbt models (no connection needed)
cd dbt/prism_dbt
dbt parse

# Run all tests against RESONATE-DEV
dbt seed && dbt run && dbt test

# Run specific test
dbt test --select test_waterfall_match
```

**Note:** All 33 dbt tests must pass against dev Snowflake before merging to main. The CI workflow runs `dbt parse` as a gate; full `dbt test` runs are manual against RESONATE-DEV.

### GitHub Actions Workflows

| Workflow | Trigger | What it does |
|---|---|---|
| `ci.yml` | PR to main | `sbt test` — runs all 443 Scala unit tests |
| `snowflake_dbt.yml` | Manual | `snow dbt deploy` — deploys prism_dbt package to Snowflake |
| `snowflake_stored_procedure.yml` | Manual | Deploys PRISM_LOOKUP + NAME_ADDRESS_LOOKUP SPs |
| `snowflake_function.yml` | Manual | Deploys NAME_ADDRESS_HASH UDF |

Workflows substitute `__PRISM_DB__` / `__PRISM_SCHEMA__` placeholders at deploy time.

---

## Spark Jobs: Key Concepts

### Data Flow

```
Raw sources (S3)
  ├── Experian ConsumerView + digital graph → ExperianPreprocessJob
  ├── L2 consumer tab-delimited            → L2ConsumerPreprocessJob
  └── ...
         ↓
  identifier_index (Parquet, partitioned by identifier_type)
  persons          (Parquet — person_id, household_id, attributes)
         ↓
  OnboardingMatchJob   → 4-phase match (blocking, corroboration, scoring, fallback)
  MergeOnboardingLinksJob → BL-30 multi-provider collapse + BL-4/BL-107 confidence scoring
  PersonIdentityJob    → main orchestrator
  CustomerMatchJob     → ephemeral 7-tier customer file matching
```

### identifier_index Schema
`(identifier_type, identifier_value, person_id, household_id, confidence, last_seen, run_date)`

Partitioned by `identifier_type`. Supported types: `HEM_MD5`, `HEM_SHA1`, `HEM_SHA256`, `MAID`, `ZIP11`, `IP_ADDRESS`, `HEM_INDIRECT`, `MAID_INDIRECT`, `HOUSEHOLD_ID`.

### Important Bug Fixes Already Applied

- **L2IndexJob HOUSEHOLD_ID null guard** — `filterNulls=true` prevents phantom household collapse.
- **OnboardingMatchJob Phase 3 tie-breaker** — `person_id.asc` added to `selectBestMatch` window for determinism.
- **ExperianPreprocessJob** — `min(id_value)` instead of `first(id_value)` for deterministic aggregation.
- **MergeOnboardingLinksJob / PersonIdentityJob** — null `identifier_value` filter before `groupBy`.
- **AddressNormalizer nickname chains** — iterative lookup (max 5 hops) for transitive nicknames.

---

## prism_dbt: Key Concepts

### 4 Primitive Macros (Consumer-Facing)

| Macro | Purpose |
|---|---|
| `waterfall_match(waterfall_order, input_column_aliases)` | Multi-step waterfall resolution: for each step in `waterfall_order`, looks up identifier in `identifier_index`, returns matched `person_id` + confidence. Routes ZIP11 to `persons.zip11`, all other types to `identifier_index`. |
| `identifier_expand(identifier_type, input_col)` | Expands one identifier type to all associated person_ids. Single identifier_type per call. |
| `persons_project(person_ids, output_columns)` | Projects person attributes from the `persons` table for a set of person_ids. |
| `lookup(identifier_type, identifier_value)` | Single-identifier debug/point lookup. Non-deterministic on confidence ties is broken by confidence + person_id. |

### Service Models (Lambda-Invokable)

Three Snowflake-native service models under `models/service/` are invokable via `EXECUTE DBT PROJECT … --select <model> --vars '<JSON>'`. Each wraps a primitive macro for Lambda-style invocation by Append, BlockGraph, Cortex, etc.

### Consumer Integration (Append v1.0)

**Supported identifier types for Append cutover:**
- `HEM_MD5`, `HEM_SHA1`, `HEM_SHA256` — HEM type selected from sibling `HEM_TYPE` column
- `MAID` — single type

**Not in default waterfall (v1.0):**
- `ZIP11` — supported by macros (routes to `persons.zip11`), but not in default waterfall until WU-26 lands
- `ip_address`, `hem_indirect`, `maid_indirect` — opt-in or obsoleted

### NAME_ADDRESS_HASH UDF

Canonical name+address → person_id hash recipe. Deployed to `RESONATE.PRISM`. Does **not** do canonical first-name lookup (`Bill` ≠ `William`) or address-abbreviation expansion in v1.0 — tracked for future versions.

---

## Project-Specific Rules and Gotchas

- **dbt `dbt_project.yml` schema:** Consumers declare their own `waterfall_order` and `input_column_aliases`. Zero changes to `prism_dbt` are needed for new consumers — just add `prism_dbt` to their `packages.yml`.
- **Spark 3 / JDK 17:** Tests require `spark.sql.legacy.allowUntypedScalaUDF=true`. The `--add-opens` JVM flags in `build.sbt` are conditional on JDK 9+ to avoid warnings.
- **StagingWriter pattern:** All heavy Spark jobs use the `StagingWriter` cache/count/write/unpersist pattern to avoid OOM on large datasets. Do not skip the `unpersist` call.
- **PRISM_LOOKUP SP:** Single-row debug tool only — not intended for batch use.
- **Snowflake deploy:** Workflows substitute `__PRISM_DB__` / `__PRISM_SCHEMA__` at deploy time. Local testing uses `RESONATE-DEV` database.
- **Design docs:** HTML files under `docs/design/` are the canonical source of truth and are published to `s3://s3.nonprod.aws.resonatedigital.net/person-identity-graph/`. To regenerate `raw_dataset_columns.html`, run `docs/scripts/build_columns_page.py` against live source headers.
- **Raw source samples:** Do NOT commit raw L2/Experian header rows from prod buckets. The `docs/README.md` explicitly warns about this.
