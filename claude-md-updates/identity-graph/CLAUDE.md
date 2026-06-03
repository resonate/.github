# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **Identity Graph** (PRISM — Person Resolution and Identity System Map) Spark pipeline for Resonate. It builds and maintains the person identity graph by joining multi-source identifier data (L2, Experian, TapAd, LiveRamp) into a unified `identifier_index` and `persons` dataset.

### Key Outputs

| Dataset | S3 Path | Description |
|---------|---------|-------------|
| `identifier_index` | `s3://resonate-core-datasets/identity-graph/` | Cross-source identifier linkages |
| `persons` | `s3://resonate-core-datasets/identity-graph/persons/` | Deduplicated person records |
| JAR | `s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar` | Published artifact |

## Repository Structure

```
├── src/main/scala/com/resonate/spark/
│   ├── apps/
│   │   ├── identity/               # Core identity graph jobs
│   │   │   ├── ExperianPreprocessJob.scala
│   │   │   ├── L2ConsumerPreprocessJob.scala
│   │   │   ├── TapadIndexJob.scala
│   │   │   ├── L2IndexJob.scala
│   │   │   ├── L2ConsumerIndexJob.scala
│   │   │   ├── OnboardingMatchJob.scala
│   │   │   ├── OnboardingHouseholdLinksJob.scala
│   │   │   ├── OnboardingPersonsJob.scala
│   │   │   ├── MergeOnboardingLinksJob.scala
│   │   │   ├── PersonIdentityJob.scala
│   │   │   ├── CustomerMatchJob.scala
│   │   │   └── ExperianDataProcessor.scala
│   │   └── l2geo/                  # L2 geo jobs (GeoLocationDaily, etc.)
│   └── utils/
│       ├── HashUtils.scala         # SHA256 person_id / household_id generation
│       ├── StagingWriter.scala     # Cache-count-write-unpersist pattern
│       ├── AddressNormalizer.scala # USPS Pub 28 suffix expansion, nickname resolution
│       ├── IpFilter.scala          # RFC1918 + datacenter IP blocklist
│       └── ScoringConfig.scala     # Confidence scoring config
├── src/test/scala/                 # Test suites (443 unit tests)
├── build.sbt                       # Scala 2.12.17, Spark 3.5.0
├── docs/
│   └── design/                     # PRISM design docs (HTML)
│       ├── design_and_production_readiness.html
│       ├── consumer_interface_design.html
│       ├── project_plan.html
│       └── raw_dataset_columns.html
└── docs/scripts/
    └── build_columns_page.py       # Regenerates raw_dataset_columns.html
```

## Common Development Tasks

### Build and Test

```bash
# Run all tests
sbt test

# Run specific test suite
sbt "testOnly com.resonate.spark.apps.identity.ExperianPreprocessJobSpec"

# Available test suites:
# ExperianPreprocessJobSpec, L2ConsumerPreprocessJobSpec, TapadIndexJobSpec
# L2IndexJobSpec, L2ConsumerIndexJobSpec, OnboardingMatchJobSpec
# OnboardingHouseholdLinksJobSpec, OnboardingPersonsJobSpec, MergeOnboardingLinksJobSpec
# PersonIdentityJobSpec, CustomerMatchJobSpec
# HashUtilsSpec, ScoringConfigSpec, SchemasSpec, ConfigSpec, IpFilterSpec
# StagingWriterSpec, AddressNormalizerSpec

# Package JAR for EMR
sbt assembly
# Published to S3 via CI: s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar
```

**JVM options (JDK 17):** The build.sbt automatically adds `--add-opens` flags for JDK 9+ to satisfy Spark 3.5 reflective access requirements.

### Deploying

Deployment follows GitOps — see [Confluence: Pipeline Scala SBT GitOps](https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/3683057840).

The CI pipeline publishes the fat JAR to:
```
s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar
```

## Key Concepts

### Pipeline Jobs

**Preprocessing:**
- `ExperianPreprocessJob` — Normalizes Experian ConsumerView + digital graph; uses `min(id_value)` for deterministic aggregation
- `L2ConsumerPreprocessJob` — Preprocesses L2 consumer tab-delimited files
- `ExperianDataProcessor` — Converts Experian offline graph deliveries (CSV.gz → Parquet); supports glob patterns

**Indexing:**
- `TapadIndexJob` — Experian/TapAd to `identifier_index` (direct + staging)
- `L2IndexJob` — L2 voter to `identifier_index`; `HOUSEHOLD_ID` null guard (filterNulls=true)
- `L2ConsumerIndexJob` — L2 consumer to `identifier_index`

**Matching:**
- `OnboardingMatchJob` — 4-phase matching (blocking, corroboration, scoring, fallback); deterministic Phase 3 tie-breaker via `person_id.asc`
- `OnboardingHouseholdLinksJob` — Household CTV/TTD attribution via digital graph
- `MergeOnboardingLinksJob` — BL-30 multi-provider collapse + BL-4/BL-107 confidence scoring

**Core:**
- `PersonIdentityJob` — Main orchestrator: persons + identifier_index from L2/LI/LR/Sovrn; null `identifier_value` guard before groupBy
- `CustomerMatchJob` — Ephemeral customer file matching (7 fallback tiers); deterministic tie-breaker

### Shared Utilities

| Utility | Purpose |
|---------|---------|
| `HashUtils` | Deterministic SHA256-based `person_id` and `household_id` generation |
| `StagingWriter` | Cache → count → write → unpersist pattern for staging partitions |
| `AddressNormalizer` | USPS Pub 28 suffix expansion, nickname resolution (up to 5 hops), soundex |
| `IpFilter` | RFC1918 + datacenter/cloud IP blocklist with IPv6 exclusion |
| `ScoringConfig` | Centralized confidence scoring, recency decay, artifact thresholds |

### PRISM Design

PRISM (Person Resolution and Identity System Map) has 6 delivery tracks:
- **Track A** — L2 ingestion
- **Track B** — Experian ingestion
- **Track C** — TapAd/LiveRamp linkage
- **Track D** — Consumer interface (Append/BlockGraph/GeoFix/Clean Room/Cortex)
- **Track E** — Operational readiness
- **Track F** — Monitoring/alerting

Design docs are published at:
`s3://s3.nonprod.aws.resonatedigital.net/person-identity-graph/`

### All Jobs Use Scopt CLI Args

All jobs use `scopt` for command-line argument parsing (no `application.conf`). Run with:
```bash
spark-submit --class com.resonate.spark.apps.identity.<JobName> \
  identity-graph-latest.jar \
  --<arg1> <value1> \
  --<arg2> <value2>
```

## Recent Changes (May–June 2026)

- **11 Scala jobs ported** from `resonate-research` into this repo (PR #22): ExperianPreprocess, L2ConsumerPreprocess, TapadIndex, L2Index, L2ConsumerIndex, OnboardingMatch, OnboardingHouseholdLinks, OnboardingPersons, MergeOnboardingLinks, PersonIdentity, CustomerMatch — 443 unit tests, all passing
- **ExperianDataProcessor** added (PR #21, CDP-118890): converts Experian gzip CSV deliveries to Parquet using Spark
- **PRISM design docs** committed to `docs/design/` (PR #25): 6 HTML artifacts backing 20 open Jira tickets (CDP-118884, 118991–119016)
- **Bug fixes applied during port:**
  - L2IndexJob `HOUSEHOLD_ID filterNulls` → true (prevents phantom household collapse)
  - OnboardingMatchJob Phase 3 non-determinism → added `person_id.asc` tie-breaker
  - ExperianPreprocessJob non-deterministic aggregation → `first()` → `min()`
  - AddressNormalizer nickname chains → iterative lookup (max 5 hops)
  - MergeOnboardingLinksJob + PersonIdentityJob → null `identifier_value` guard before groupBy
