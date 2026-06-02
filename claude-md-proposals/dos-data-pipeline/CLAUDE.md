# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**dos-data-pipeline** contains the Scala/Spark data pipeline jobs for Resonate's DOS (Data Operations & Services) platform. The primary pipelines are `geo-location` (IP → postal/district mapping) and `segment-aggregator`. Other pipelines (ToBitmap, BitmapEnrichment, etc.) are triggered by the `datapipeline-trigger` Lambda in `lambda-modeling`; their configurations live in `services-api-configuration`.

## Tech Stack

- **Scala** / **Spark** (EMR 6.x or 7.x depending on pipeline)
- **SBT** build tool
- **GitHub Actions** for CI/CD deployment
- Configs for geo-location and segment-aggregator live **in this repo** (deployed by GitHub Actions)

## Repository Structure

```
├── src/main/scala/com/resonate/
│   ├── geo/                     # GeoLocationDaily, GeoLocationFullBackfill, GeoLocationFull
│   ├── segment/                 # SegmentAggregator
│   ├── bitmap/                  # ToBitmap (legislative-district marker bits)
│   └── utils/                   # DistrictResolver, shared helpers
├── src/test/scala/com/resonate/
│   └── ...                      # Unit tests
└── config/                      # Geo-location and segment-aggregator configs (deployed in-repo)
```

## Common Development Tasks

### Build and Test

```bash
# Run all unit tests
sbt test

# Run a specific test suite
sbt "testOnly com.resonate.geo.GeoLocationDailyTest"

# Package the JAR
sbt assembly
```

### CI/CD Deploy

Deployment is through GitHub Actions. Configuration files for geo-location and segment-aggregator in this repo are deployed by Actions — **do not apply config changes manually to S3 in dev/prod**.

## Key Jobs

### GeoLocationDaily

Processes daily RcidEnriched output — appends `zip`, `congress`, `state_senate`, `state_house`, `proposed_congress`, `district_source`, and `l2_party_confirmed` columns.

**DistrictResolver** (CDP-118946): centralises L2-vs-IP district coalescing:

| `district_source` | Condition |
|---|---|
| `L2_CONFIRMED` | L2 has district AND `l2_party_confirmed = true` |
| `L2_UNCONFIRMED` | L2 has district AND `l2_party_confirmed != true` |
| `IP_INFERRED` | No L2 district, but IP zip lookup populated at least one district |
| `null` | No L2 and no IP match |

The 4-file zip→district mapping (`zip-congress-mapping.csv`, `zip-proposed-congress-mapping.csv`, `zip-state-senate-mapping.csv`, `zip-state-house-mapping.csv`) is passed via a single base-path arg `zipDistrictMappingsBasePath`. Files are read by convention, not hardcoded — do not rename the CSV files.

### GeoLocationFullBackfill

Re-derives all four districts for every RCID in the rolling-full. Reads `zip` from the existing fullDf (populated at Daily time) — does NOT make a second NetAcuity lookup.

**Key invariant**: `districtColumnList` arg was removed (CDP-118946). The Backfill always re-derives all four districts. The orchestrator's `geo_location.asl.json` must NOT pass `DistrictColumnList`.

### ToBitmap (CDP-118944 / CDP-118947)

Sets the `A200299998` (L2-confirmed) marker bit. The marker is:
- Set unconditionally (Phase A) — present on every RCID
- Will be narrowed to `district_source = L2_CONFIRMED` in Phase 2 Unit 5

**Reserved bit range**: `[100300001, 200300000]`. ToBitmap's `mergeGeoAttrsForRoaringBitmapUdf` wipes `[100300001, 200299999)` on every run — `200299998` is inside this range. Do not assign new geo bits outside `[200299998, 200300000]`.

### l2_party_confirmed (CDP-118945)

Added to `rcid-l2-enriched` output (via `core-data-applications/L2HemToRcid`):

- `true`: cookie party null, L2 party non-major, or parties match
- `false`: both major (Dem/Rep) and disagree (Bucket 2, ~10% of cookie jar)

This field gates the `district_source` classification downstream.

## Coordinated Deploys

The following units must deploy together in a single window:
- **Unit 3** (`core-data-applications`): `l2_party_confirmed` from `L2HemToRcid`
- **Unit 4** (`dos-data-pipeline`): `district_source` + IP fill in `GeoLocationDaily`/`GeoLocationFullBackfill`
- **Unit 5** (`dos-data-pipeline`): `ToBitmap` market-bit gating on `L2_CONFIRMED`

Merging any one without the others leaves the pipeline in a broken intermediate state.

## Testing Guidelines

- `GeoLocationDailyTest`: rewires fixtures into `ZipDistrictMappings`; exercises DistrictResolver paths directly
- `GeoLocationFullBackfillTest`: cover all 7 `district_source` paths (L2_CONFIRMED, L2_UNCONFIRMED, IP_INFERRED, null, partial-L2, IP-only, no-match)
- `GeoLocationFullTest`: update tuple arities when new output columns are added
- `sbt test` must pass before any PR; 60 tests as of 2026-05-20

## Key Constraints

- **Never pass `DistrictColumnList` to `GeoLocationFullBackfill`** — the arg was removed in CDP-118946
- **Never make a second NetAcuity lookup in Backfill** — reuse `zip` from fullDf
- **4-file zip mapping must stay named exactly** as listed — read by convention
- **Marker bit `200299998` must stay inside the wipe range** `[100300001, 200299999)`
- Branch naming: `feature/{JIRA-ID}-{kebab-slug}`
