# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**identity-graph** contains the Scala/Spark jobs that build Resonate's person identity graph — linking RCIDs to hashed emails, phone numbers, voter records, Experian digital graph data, and Tapad device IDs. It runs on AWS EMR via step functions in `step-function-workflow-orchestrator`.

## Tech Stack

- **Scala** 2.12.17 / **Spark** 3.5.0
- **SBT** build tool
- **JDK 17** runtime (--add-opens flags required — already configured in `build.sbt`)
- **Scopt** for CLI argument parsing (all jobs use scopt, not `application.conf`)
- Deploy: JAR published to `s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar`

## Repository Structure

```
├── src/main/scala/com/resonate/
│   ├── jobs/                    # 11 pipeline jobs (see catalogue below)
│   └── utils/                   # Shared utilities
│       ├── HashUtils             # Deterministic SHA256-based person_id / household_id generation
│       ├── StagingWriter         # cache → count → write → unpersist pattern
│       ├── AddressNormalizer     # USPS Pub 28 suffix expansion, nickname resolution (transitive, max 5 hops)
│       ├── IpFilter              # RFC1918 + datacenter/cloud IP blocklist
│       └── ScoringConfig         # Centralized confidence scoring and recency decay
├── src/test/scala/com/resonate/
│   └── ...                      # 443 unit tests across 20 test suites
└── build.sbt
```

## Common Development Tasks

### Build and Test

```bash
# Run all unit tests
sbt test

# Run a specific test suite
sbt "testOnly com.resonate.jobs.PersonIdentityJobSpec"

# Package the JAR for EMR deployment
sbt assembly
# Output: target/scala-2.12/identity-graph-assembly-*.jar

# AWS authentication (needed before uploading JARs)
aws sso login
```

### CI/CD Deploy

The JAR is automatically built and uploaded to S3 on push to `main`. The GitOps flow is documented at the Confluence link in the README. For feature branch testing, manually dispatch the CI publish workflow with `environment=dev`.

## Job Catalogue

| Job | Purpose |
|---|---|
| `ExperianPreprocessJob` | Normalises Experian ConsumerView + digital graph (CSV/Parquet → normalised Parquet) |
| `L2ConsumerPreprocessJob` | Preprocesses L2 consumer tab-delimited files |
| `TapadIndexJob` | Maps Experian/Tapad identifiers to `identifier_index` (direct + staging) |
| `L2IndexJob` | Maps L2 voter records to `identifier_index` entries |
| `L2ConsumerIndexJob` | Maps L2 consumer records to `identifier_index` entries |
| `OnboardingMatchJob` | 4-phase matching: blocking → corroboration → scoring → fallback |
| `OnboardingHouseholdLinksJob` | Household CTV/TTD attribution via digital graph |
| `OnboardingPersonsJob` | Creates Tapad-only person records (deprecated, BL-106) |
| `MergeOnboardingLinksJob` | Multi-provider collapse + BL-4/BL-107 confidence scoring |
| `PersonIdentityJob` | Main orchestrator: persons + identifier_index from L2/LI/LR/Sovrn |
| `CustomerMatchJob` | Ephemeral customer file matching (7 fallback tiers) |

## Key Invariants and Bug Fixes

These are non-obvious correctness constraints — always preserve them:

- **`L2IndexJob` HOUSEHOLD_ID**: `filterNulls = true` (not false) — prevents phantom household collapse from null household IDs.
- **`OnboardingMatchJob` Phase 3**: Window for `selectBestMatch` includes `person_id.asc` tie-breaker — prevents non-deterministic output.
- **`ExperianPreprocessJob` aggregation**: Use `min(id_value)` not `first(id_value)` — prevents non-deterministic aggregation.
- **`AddressNormalizer` nickname chains**: `resolveNicknameUDF` applies iterative lookup (max 5 hops) for transitive chains.
- **`MergeOnboardingLinksJob` / `PersonIdentityJob`**: null `identifier_value` filter before groupBy prevents null-key groupBy errors.

## Shared Utilities

### HashUtils
Generates deterministic `person_id` and `household_id` via SHA256. Do not change the hashing logic — it must be stable across runs.

### StagingWriter
Wraps the `cache → count → write → unpersist` pattern for staging partitions. Use this instead of ad-hoc caching.

### AddressNormalizer
Implements USPS Pub 28 suffix expansion and nickname resolution. Nickname chains are resolved transitively (up to 5 hops). The `ip_blocklist.txt` should be refreshed periodically.

### ScoringConfig
Centralises confidence scoring thresholds, recency decay parameters, and artifact thresholds. Do not hardcode scoring values in individual jobs.

## Testing Guidelines

- Each job MUST have a corresponding `*Spec.scala` test suite
- All Spark tests use `SparkSession` in local mode — no external dependencies
- JDK 17 requires the `--add-opens` flags already present in `build.sbt`; do NOT remove them
- Test fixtures use column-aligned parquet schemas — verify row counts match expected values
- `parallelExecution in Test := false` — tests run sequentially to avoid Spark session conflicts

## Integration with step-function-workflow-orchestrator

This repo is the Spark side only. The companion Step Function pipelines live in `step-function-workflow-orchestrator`:
- `pipelines/experian-data-processing/` — runs `ExperianPreprocessJob` + `ExperianDataProcessor`
- `pipelines/person-identity-graph/` — runs the full identity pipeline sequence

Changing a job's CLI args requires a coordinated PR in both repos.

## Key Constraints

- **Never change SHA256 hashing logic in `HashUtils`** — any change breaks historical linkage
- **Never use `first()`** for aggregations where order is not guaranteed — use `min()` or `max()`
- **Null guards before groupBy** — always filter null identifier_value/household_id before groupBy
- Branch naming: `feature/{JIRA-ID}-{kebab-slug}`
