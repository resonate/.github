# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **Identity Graph** Scala/Spark pipeline — a collection of 11 Spark jobs that build Resonate's person identity graph from multiple data sources (Experian, L2, Tapad, LiveIntent, LiveRamp, Sovrn). It also contains the **ExperianDataProcessor** which converts Experian offline delivery files from gzip CSV to Parquet.

Deployment docs: https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/3683057840/Pipeline+Scala+SBT+GitOps

**Production JAR:** `s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar`

## Tech Stack

- **Scala** 2.12.17
- **Spark** 3.5.0
- **SBT** build tool
- **AWS EMR** for execution

## Repository Structure

```
├── src/main/scala/com/resonate/
│   ├── identitygraph/
│   │   ├── jobs/                       # 11 Spark pipeline jobs
│   │   │   ├── ExperianPreprocessJob   # Normalizes Experian ConsumerView + digital graph
│   │   │   ├── L2ConsumerPreprocessJob # Preprocesses L2 consumer tab-delimited files
│   │   │   ├── TapadIndexJob           # Experian/Tapad → identifier_index
│   │   │   ├── L2IndexJob              # L2 voter → identifier_index
│   │   │   ├── L2ConsumerIndexJob      # L2 consumer → identifier_index
│   │   │   ├── OnboardingMatchJob      # 4-phase matching (blocking, corroboration, scoring, fallback)
│   │   │   ├── OnboardingHouseholdLinksJob  # Household CTV/TTD attribution via digital graph
│   │   │   ├── OnboardingPersonsJob    # Creates Tapad-only person records (deprecated BL-106)
│   │   │   ├── MergeOnboardingLinksJob # BL-30 multi-provider collapse + confidence scoring
│   │   │   ├── PersonIdentityJob       # Main orchestrator: persons + identifier_index
│   │   │   └── CustomerMatchJob        # Ephemeral customer file matching (7 fallback tiers)
│   │   └── utils/                      # Shared utilities
│   │       ├── HashUtils               # SHA256-based person_id and household_id generation
│   │       ├── StagingWriter           # Cache, count, write, unpersist pattern
│   │       ├── AddressNormalizer       # USPS Pub 28 suffix expansion, nickname resolution
│   │       ├── IpFilter                # RFC1918 + datacenter/cloud IP blocklist
│   │       └── ScoringConfig           # Centralized confidence scoring, recency decay
│   └── experian/
│       └── ExperianDataProcessor       # Converts Experian gzip CSV → Parquet
├── src/test/scala/com/resonate/
│   └── identitygraph/                  # 443 unit tests in 20 test suites
├── build.sbt
└── README.md
```

## Common Development Tasks

### Build and Test

```bash
# Compile
sbt compile

# Run all tests (443 tests across 20 suites)
sbt test

# Run a specific test suite
sbt "testOnly com.resonate.identitygraph.jobs.PersonIdentityJobSpec"

# Package JAR for EMR
sbt assembly
# Output: target/scala-2.12/identity-graph-assembly-*.jar

# AWS auth (required for S3 access)
aws sso login
```

### JVM Memory (CI)

The SBT build is configured with 3GB heap and 1.5GB forked test JVM. On JDK 17 the `--add-opens` flags are automatically applied via `build.sbt`. Do not change these settings without testing on CI.

### Deploying

Deployment follows the GitOps process described in the Confluence page above. The JAR is built and uploaded to S3, then referenced by the `identity-graph` Step Function in `step-function-workflow-orchestrator`.

## Key Concepts

### Person Identity Pipeline (11 Jobs)

The pipeline processes multiple identity sources and produces `identifier_index` entries:

| Job | Input | Purpose |
|-----|-------|---------|
| ExperianPreprocessJob | Experian CSV gzip | Normalize ConsumerView + digital graph |
| L2ConsumerPreprocessJob | L2 tab-delimited | Preprocess L2 consumer data |
| TapadIndexJob | Experian/Tapad parquet | Create identifier_index (direct + staging) |
| L2IndexJob | L2 voter | Create identifier_index; null guard on HOUSEHOLD_ID |
| L2ConsumerIndexJob | L2 consumer | Create identifier_index entries |
| OnboardingMatchJob | All sources | 4-phase match; Phase 3 uses person_id.asc tie-breaker |
| OnboardingHouseholdLinksJob | Digital graph | Household CTV/TTD attribution |
| OnboardingPersonsJob | Tapad-only | Person records (deprecated) |
| MergeOnboardingLinksJob | Onboarding links | Multi-provider collapse, BL-4/BL-107 confidence |
| PersonIdentityJob | All | Main orchestrator |
| CustomerMatchJob | Customer file | 7-tier fallback matching |

### HashUtils

- `personId(seed)` — deterministic SHA256-based person_id
- `householdId(seed)` — deterministic SHA256-based household_id

Seeds are constructed from PII fields (name, address, etc.) after normalization.

### AddressNormalizer

- Applies USPS Publication 28 suffix expansions (e.g., `ST → STREET`)
- Resolves nickname chains iteratively (max 5 hops, e.g., `BOB → ROBERT → BOB` is cycle-safe)

### ExperianDataProcessor

Converts Experian offline delivery gzip CSV files to Parquet:
- Subtables: `consumerview`, `email`, `phone`
- Supports glob patterns for input paths (e.g., `Resonate.ConsumerView_*.gz`)
- Digital graph is already Parquet — no conversion needed

## Recent Changes (as of 2026-05)

- **PR #22**: Ported all 11 Person Identity Graph Scala jobs from resonate-research; 443 unit tests all passing
  - Fixed `L2IndexJob` HOUSEHOLD_ID null guard (`filterNulls = true`)
  - Fixed `OnboardingMatchJob` Phase 3 non-determinism (added `person_id.asc` tie-breaker)
  - Fixed `ExperianPreprocessJob` non-deterministic aggregation (`min(id_value)` vs `first`)
  - Fixed `AddressNormalizer` nickname chains (iterative lookup, max 5 hops)
  - Added null guards in `MergeOnboardingLinksJob` and `PersonIdentityJob`
- **PR #21**: Added `ExperianDataProcessor` Spark app for gzip CSV → Parquet conversion

## Testing Guidelines

All 11 jobs have dedicated test suites. Key invariants to maintain:
- `L2IndexJob`: `filterNulls = true` for HOUSEHOLD_ID to prevent phantom household collapse
- `OnboardingMatchJob` Phase 3: always include `person_id.asc` as a tie-breaker in `selectBestMatch` window
- `ExperianPreprocessJob`: use `min(id_value)` not `first()` for deterministic aggregation
- `MergeOnboardingLinksJob` / `PersonIdentityJob`: filter null `identifier_value` before groupBy
