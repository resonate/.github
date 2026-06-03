# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **Data-of-Scalars (DOS) Data Pipeline** — a Scala/Spark pipeline that processes geographic and district data for Resonate's political audience targeting product. It includes the geo-location pipeline (`GeoLocationDaily`, `GeoLocationFullBackfill`) and segment aggregation jobs.

### Key Outputs

| Dataset | S3 Path |
|---------|---------|
| Daily geo | `s3://resonate-core-datasets/data/geo/daily/<YYYYMMDD>/` |
| Full geo | `s3://resonate-core-datasets/data/geo/full/<YYYYMMDD>/` |
| Legislative geo | `s3://resonate-core-datasets/data/geo/legislative/full/<YYYYMMDD>/` |
| Bitmap output | Used by ToBitmap / segment-aggregator downstream |

## Repository Structure

```
├── src/main/scala/com/resonate/spark/apps/
│   ├── GeoLocationDaily.scala          # Daily geo enrichment job
│   ├── GeoLocationFullBackfill.scala   # Full backfill job
│   ├── GeoLocationFull.scala           # Rolling-full maintenance
│   └── ToBitmap.scala                  # Converts geo data to bitmap format
├── src/test/scala/                     # Test suites
├── build.sbt                           # Scala 2.11.8, Spark 2.4.3
└── pipelines/
    ├── geo-location/
    │   └── config/
    │       └── zip-district-mappings/  # ZIP→district CSV mapping files
    │           ├── zip-congress-mapping.csv
    │           ├── zip-proposed-congress-mapping.csv
    │           ├── zip-state-senate-mapping.csv
    │           └── zip-state-house-mapping.csv
    └── segment-aggregator/
```

## Common Development Tasks

### Build and Test

```bash
# Compile
sbt compile

# Run all tests
sbt test

# Run specific test
sbt "testOnly com.resonate.spark.apps.GeoLocationDailyTest"

# Package for EMR
sbt assembly
```

**Note:** This project uses **Scala 2.11.8 / Spark 2.4.3** (older than the identity-graph repo).

### Deploying

Deployment is via GitHub Actions. See `.github/workflows/` for the deployment workflow targeting the `dos-data-pipeline` EMR job.

### Updating ZIP→District Mappings

The 4 CSV files under `pipelines/geo-location/config/zip-district-mappings/` are deployed to:
```
s3://resonate-core-applications{-env}/dos/geo-location/configs/zip-district-mappings/
```

They are versioned together as a snapshot and must use **L2 canonical namespace** format (e.g., `NHBELKNAP-01`, not `NH001`).

**Schema:** Each file has columns `zip,state,<district-type>`.

## Key Concepts

### District Source Provenance (`district_source`)

Every geo record now carries a `district_source` field indicating how the district was resolved:

| Value | Meaning |
|-------|---------|
| `L2_CONFIRMED` | L2 has a district AND `l2_party_confirmed = true` |
| `L2_UNCONFIRMED` | L2 has a district AND `l2_party_confirmed != true` |
| `IP_INFERRED` | No L2 district but zip→district mapping resolved via IP-geolocation |
| `null` | No district data available |

Use `DistrictResolver` (the centralized helper) for all `district_source` derivation — do not duplicate the `when`/`coalesce` ladder.

### IP-Inferred District Fallback

`GeoLocationDaily` and `GeoLocationFullBackfill` both support IP-inferred district fallback:
- Uses the 4 ZIP→district CSV files as a lookup
- `GeoLocationDaily` resolves ZIP from NetAcuity IP lookup
- `GeoLocationFullBackfill` reads existing `zip` from the rolling-full (no second IP lookup needed)

### ToBitmap Gating

The L2-confirmed marker bit in ToBitmap is gated on `district_source = "L2_CONFIRMED"`. Do NOT set the marker unconditionally — `L2_UNCONFIRMED` and `IP_INFERRED` rows must not trigger the confirmed marker.

### ZIP→District CSV Namespace Requirements

The mapping CSVs **must** use L2 canonical district codes (matching `DistrictToEvkey.csv`):
- State house: county-based format, e.g., `NHBELKNAP-01` (NOT floterial `NH001`)
- State senate: L2 descriptive format, e.g., `MABRISTOL 14` (NOT floterial `MA023`)
- Congress: numeric, e.g., `MA01`

Using floterial codes will cause rows to be silently dropped at the `state-legislative-districts-aggregation` join.

**States requiring county-based namespace (state house):** NH, MA, MN, VT, MD, SD, ND
**States requiring L2 canonical namespace (state senate):** MA, AK, VT, DC, CT

### GeoLocationFullBackfill

- Does NOT take `districtColumnList` argument — always re-derives all 4 districts (congress, proposed_congress, state_senate, state_house) from scratch
- Reads `zip` from existing fullDf to run IP-fallback without a second NetAcuity lookup
- `GeoLocationFull.expectedCols` includes `zip` and `district_source`

## Recent Changes (May–June 2026)

- **IP-inferred district fallback** (PR #103, CDP-118946): Added `DistrictResolver` helper; both `GeoLocationDaily` and `GeoLocationFullBackfill` now support IP-inferred district lookup via 4 ZIP→district CSVs. Observed distribution: IP_INFERRED ~86.8%, L2_CONFIRMED ~9.9%, L2_UNCONFIRMED ~2.9%, null ~0.3%
- **`district_source` provenance** (PR #103): New column in geo output encoding how each district was resolved
- **ToBitmap L2-confirmed gating** (PR #103, CDP-118947): Marker bit now requires `district_source = "L2_CONFIRMED"` (was unconditional)
- **4-file ZIP→district mapping** (PR #103): Split consolidated CSV into 4 separate type-specific files with `zipDistrictMappingsBasePath` argument replacing `DistrictColumnList`
- **ZIP→district namespace fix — state house** (CDP-118946): Rewrote NH (PR #2 in step-function-workflow-orchestrator), then MA/MN/VT/MD/SD/ND (PR #3) from floterial → L2 canonical namespace; 0 unmatched state-house codes after fix
- **ZIP→district namespace fix — state senate** (CDP-118946): Rewrote MA/AK/VT/DC state senate codes; 2 remaining unmatched (stale data: `AK11`, `CT37`)
- **`state_senate`/`state_house` raw concat revert** (PR #15 in core-data-applications, CDP-118951): Reverted `parseDistrict` back to `concat(state, district)` for state-leg — numeric parsing was valid only for congress codes, not state-leg
