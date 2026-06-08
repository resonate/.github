# identity-graph

This repo contains Resonate's **PRISM Identity Graph** — Scala/Spark ETL jobs that build and maintain the person identity graph, plus the **`prism_dbt` v1.0** Snowflake-native dbt package that exposes PRISM as a consumer-facing service layer.

## Deployment

Final JAR path in production: `s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar`

See the [Pipeline Scala SBT GitOps runbook](https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/3683057840/Pipeline+Scala+SBT+GitOps) for full deployment steps.

## Build

**JDK 17 is required.**

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home
sbt compile
sbt test        # 443+ unit tests across 20+ suites
sbt assembly    # fat JAR for EMR → target/identity-graph-assembly-*.jar
```

JDK 17 requires `--add-opens` flags for Spark 3.5 tests. These are wired into `build.sbt` conditionally for JDK 9+.

## Repository Structure

```
src/main/scala/com/resonate/spark/apps/
  sampling/                  # CookieJarSampler (DEPRECATED — do not run)
  mapid/                     # MapIdToRid and related jobs
  identity/
    ExperianPreprocessJob    # Normalizes Experian ConsumerView + digital graph
    L2ConsumerPreprocessJob  # Preprocesses L2 consumer tab-delimited files
    TapadIndexJob            # Experian/Tapad → identifier_index
    L2IndexJob               # L2 voter → identifier_index entries
    L2ConsumerIndexJob       # L2 consumer → identifier_index entries
    OnboardingMatchJob       # 4-phase matching (blocking/corroboration/scoring/fallback)
    OnboardingHouseholdLinksJob # Household CTV/TTD attribution via digital graph
    OnboardingPersonsJob     # Creates Tapad-only person records (DEPRECATED BL-106)
    MergeOnboardingLinksJob  # BL-30 multi-provider collapse + BL-4/BL-107 confidence scoring
    PersonIdentityJob        # Main orchestrator: persons + identifier_index
    CustomerMatchJob         # Ephemeral customer file matching (7 fallback tiers)

src/main/scala/com/resonate/spark/utils/
  HashUtils          # Deterministic SHA256-based person_id / household_id generation
  StagingWriter      # Cache, count, write, unpersist pattern for staging partitions
  AddressNormalizer  # USPS Pub 28 suffix expansion, nickname resolution, soundex
  IpFilter           # RFC1918 + datacenter/cloud IP blocklist with IPv6 exclusion
  ScoringConfig      # Centralized confidence scoring, recency decay, artifact thresholds

dbt/
  prism_dbt/         # prism_dbt v1.0 — Snowflake-native consumer dbt package
    macros/          # 4 primitive macros: waterfall_match, identifier_expand, persons_project, lookup
    models/service/  # 3 Lambda-invokable service models (waterfall, identifier_expand, persons_project)
    seeds/           # Test seed data
    tests/           # Unit + singular + schema tests (33 total)

.github/workflows/
  snowflake_dbt.yml              # Deploy prism_dbt package via `snow dbt deploy`
  snowflake_stored_procedure.yml # Deploy PRISM_LOOKUP + NAME_ADDRESS_LOOKUP SPs
  snowflake_function.yml         # Deploy NAME_ADDRESS_HASH UDF
```

## Person Identity Graph Jobs

The 11 `identity/` jobs build PRISM's core tables (`persons`, `identifier_index`, `identifier_graph`) from source data (L2, Experian/Tapad, LiveIntent, Sovrn):

| Job | Input | Output |
|-----|-------|--------|
| `ExperianPreprocessJob` | Experian ConsumerView + digital graph parquet | Normalized Experian parquet |
| `L2ConsumerPreprocessJob` | L2 consumer tab-delimited files | Preprocessed parquet |
| `TapadIndexJob` | Experian/Tapad data | `identifier_index` entries (direct + staging) |
| `L2IndexJob` | L2 voter files | `identifier_index` entries |
| `L2ConsumerIndexJob` | L2 consumer files | `identifier_index` entries |
| `OnboardingMatchJob` | Multi-source identifier data | Matched person links (4-phase) |
| `OnboardingHouseholdLinksJob` | Digital graph + household data | Household CTV/TTD attribution |
| `MergeOnboardingLinksJob` | Multi-provider links | Collapsed + scored person links |
| `PersonIdentityJob` | L2/LI/LR/Sovrn sources | `persons` + `identifier_index` tables |
| `CustomerMatchJob` | Ephemeral customer files | Matched customer records (7 tiers) |

## prism_dbt v1.0 (Snowflake-native)

`prism_dbt` is the consumer-facing dbt package that exposes PRISM as a service. It is deployed to Snowflake and invoked by Lambda functions via `EXECUTE DBT PROJECT`.

### 4 primitive macros

- **`waterfall_match`** — resolves an input identifier through PRISM's waterfall order; supports `HEM_MD5`, `HEM_SHA1`, `HEM_SHA256`, `MAID` (ZIP11 supported but not in v1.0 default waterfall — pending WU-26)
- **`identifier_expand`** — expands a person_id to all known identifiers of a given type
- **`persons_project`** — projects person attributes from the persons table
- **`lookup`** — single-row debug lookup with deterministic tiebreak (confidence → source_count → max_effective_weight → person_id)

### Service models (Lambda-invokable)

Three models in `models/service/` are materialized on demand via `EXECUTE DBT PROJECT … --select <model> --vars '<JSON>'`:

- `waterfall` — waterfall match result
- `identifier_expand` — identifier expansion result
- `persons_project` — persons projection result

### Snowflake objects deployed

- `NAME_ADDRESS_HASH` UDF (`RESONATE.PRISM`) — canonical name+address → person_id hash
- `PRISM_LOOKUP` SP — single-row debug lookup by identifier
- `NAME_ADDRESS_LOOKUP` SP — resolve by raw PII

### Deploy workflows

All deploy workflows substitute `__PRISM_DB__` / `__PRISM_SCHEMA__` placeholders at deploy time:

```bash
# Deploy prism_dbt package (runs dbt parse gate first)
gh workflow run snowflake_dbt.yml

# Deploy stored procedures (PRISM_LOOKUP + NAME_ADDRESS_LOOKUP)
gh workflow run snowflake_stored_procedure.yml

# Deploy NAME_ADDRESS_HASH UDF
gh workflow run snowflake_function.yml
```

### Testing prism_dbt

```bash
cd dbt/prism_dbt

# Install dbt (pinned versions)
pip install dbt-core==1.11.2 dbt-snowflake==1.11.1 --only-binary :all:

# Run full test suite against RESONATE-DEV (33 tests)
dbt seed
dbt run
dbt test
```

## Key Rules and Gotchas

- **Confidence scoring uses `ScoringConfig`** — do not hard-code thresholds inline. All artifact thresholds, recency decay, and confidence weights live in `ScoringConfig`.
- **`AddressNormalizer` nickname chains** — resolveNicknameUDF applies iterative lookup up to 5 hops. Single-pass nickname resolution produces incorrect results for transitive chains (e.g. Bill → William → Will).
- **`L2IndexJob` HOUSEHOLD_ID null guard** — `filterNulls = true` on HOUSEHOLD_ID prevents phantom household collapse. Do not set to false.
- **`OnboardingMatchJob` Phase 3 tie-breaker** — `person_id.asc` tie-breaker is required in `selectBestMatch` to ensure deterministic output. Non-deterministic aggregation produces different results across runs.
- **`MergeOnboardingLinksJob` and `PersonIdentityJob` null guards** — null `identifier_value` filter before `groupBy` is required to avoid incorrect grouping.
- **ZIP11 in `prism_dbt`** — ZIP11 routing infrastructure is in place but ZIP11 is NOT in the default `append_waterfall_order` for v1.0 (pending WU-26 / `persons.zip11` column). Re-enabling is one line in `dbt_project.yml` once WU-26 lands.
- **`OnboardingPersonsJob` is deprecated** (BL-106) — kept for reference but should not be run.
- **QUERY_TAG observability** — all prism_dbt service model invocations set `QUERY_TAG` in `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` in the format `prism:<model>:<consumer>:<job_run_id>`.
- **IP blocklist is static** — `ip_blocklist.txt` refresh should be tracked as a scheduled ticket. The current file covers RFC1918 + known datacenter/cloud ranges with IPv6 exclusion.
