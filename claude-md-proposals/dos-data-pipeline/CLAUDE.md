# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **DOS Data Pipeline** — Scala/Spark jobs that process data for the Data Operations/Science (DOS) team. The primary pipelines are:

- **GeoLocationDaily** — Enriches daily RcidEnriched with geo-location data (congressional districts, state legislative districts) using L2 data and IP-inferred fallback
- **GeoLocationFull** — Maintains the rolling geo-location full dataset
- **GeoLocationFullBackfill** — Backfills geo-location data across a date range using daily-on-multi-day-pixels approach
- **ToBitmap** — Converts geo-location data to bitmap format, gating the L2-confirmed marker bit on `district_source`

## Tech Stack

- **Scala** 2.11.8
- **Spark** 2.4.3 (deployed on EMR via `dos-data-pipeline-latest.jar`)
- **SBT** build tool
- **GitHub Actions CI** with SonarCloud integration

## Repository Structure

```
├── src/main/scala/com/resonate/
│   ├── geolocation/
│   │   ├── GeoLocationDaily        # Daily enrichment job
│   │   ├── GeoLocationFull         # Rolling full dataset maintenance
│   │   ├── GeoLocationFullBackfill # Date-range backfill (daily-on-multi-day-pixels)
│   │   ├── DistrictResolver        # L2-vs-IP district coalescing + district_source derivation
│   │   └── ToBitmap                # Geo → bitmap conversion (gates L2-confirmed bit)
│   └── ...
├── src/test/scala/com/resonate/
│   └── ...                         # Unit test suites (60 tests)
├── build.sbt
└── README.md
```

## Common Development Tasks

### Build and Test

```bash
# Compile
sbt compile

# Run all tests
sbt test

# Run specific test
sbt "testOnly com.resonate.geolocation.GeoLocationDailyTest"

# Package JAR for EMR deployment
sbt assembly

# AWS auth (required for S3 access)
aws sso login
```

### Deploying

The JAR is published to S3 and consumed by the `geo-location` Step Function in `step-function-workflow-orchestrator`. Configuration (zip→district CSVs, params.json) is deployed via Terragrunt.

For geo-location and segment-aggregator, configuration is sourced from this repository. For other pipelines, configuration comes from services-api-configuration via the datapipeline-trigger Lambda.

## Key Concepts

### DistrictResolver

Central helper for deriving district data from two sources: L2 voter file and IP-inferred (NetAcuity) zip→district mappings.

**district_source values:**
| Value | Condition |
|-------|-----------|
| `L2_CONFIRMED` | L2 has any district AND `l2_party_confirmed = true` |
| `L2_UNCONFIRMED` | L2 has any district AND `l2_party_confirmed != true` |
| `IP_INFERRED` | No L2 district but zip mapping produced a result |
| `null` | No district data available |

**Zip→District Mappings (4-file snapshot):**
Deployed to `s3://resonate-core-applications{-env}/dos/geo-location/configs/zip-district-mappings/`:
- `zip-congress-mapping.csv` — L2 mode-per-zip, all 51 states
- `zip-proposed-congress-mapping.csv` — 2026 redistricted file (CA/MO/NC/OH/TX/UT/VA)
- `zip-state-senate-mapping.csv` — L2 mode-per-zip
- `zip-state-house-mapping.csv` — L2 mode-per-zip

Schema: `zip,state,<district-type>`. Files version together as a snapshot passed via `myZipDistrictMappingsBasePath`.

### ToBitmap

Gates the L2-confirmed marker bit on `district_source = "L2_CONFIRMED"`. Rows with `L2_UNCONFIRMED` or `IP_INFERRED` are enriched with district data but do not set the confirmed bit.

### GeoLocationFullBackfill

Rewrote to use a **daily-on-multi-day-pixels** approach:
- Takes a date-range glob from the orchestrator (rendered `$.ParamsConfig.Rendered.myZipDistrictMappingsBasePath`)
- Always re-derives all four districts (does not take a `DistrictColumnList` arg)
- Reads existing `zip` from fullDf for IP-fallback path (no second NetAcuity lookup)

### GeoLocation Should Run Full (Orchestrator Fix)

The `Should Run Full` choice state in `geo_location.asl.json` checks `IsPresent` on `$.FullInputPath` before the path-equality fallback. This ensures backfill output is preserved when today's full already exists (fixes CDP-118512).

## Recent Changes (as of 2026-05)

- **CDP-118946 (PR #103)**: Added IP-inferred district fallback and `district_source` provenance
  - New `DistrictResolver` helper centralizes L2-vs-IP coalescing
  - 4-file zip→district mappings split by district type
  - `GeoLocationFullBackfill` rewritten to daily-on-multi-day-pixels approach
  - `GeoLocationDaily.extract` and `GeoLocationFullBackfill.fullBackfill` both use DistrictResolver
  - `GeoLocationFull.expectedCols` extended with `zip` and `district_source`
- **CDP-118947 (PR #103)**: `ToBitmap` gates L2-confirmed marker bit on `district_source = "L2_CONFIRMED"`
- **CDP-118944**: Added `ToBitmap` unit tests for district_source gating

## Testing Guidelines

- `GeoLocationDailyTest`: test fixtures use `ZipDistrictMappings`; DistrictResolver invoked directly
- `GeoLocationFullBackfillTest`: 7 coverage cases for each `district_source` path (L2_CONFIRMED, L2_UNCONFIRMED, IP_INFERRED, null, partial-L2 coalesce, IP-only, no-match)
- `GeoLocationFullTest`: tuple arities updated for `zip` and `district_source` columns
- When adding a new district type, add entries to all 4 mapping files and update `DistrictResolver` coalescing logic
- `sort_array` must be applied to state-leg candidate arrays before hash-based pick to ensure determinism

## Configuration

**Configuration is deployed from this repo** (unlike other pipelines that use services-api-configuration). The `geo-location` pipeline reads its config from `s3://resonate-core-applications{-env}/dos/geo-location/configs/`.

When modifying configuration:
1. Update files in `src/main/resources/` or the appropriate config directory
2. The deploy pipeline (in `step-function-workflow-orchestrator`) uploads them to S3 via Terragrunt
