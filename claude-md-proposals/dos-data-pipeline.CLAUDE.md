# CLAUDE.md — resonate/dos-data-pipeline
# This is a NEW file to be created at the root of the dos-data-pipeline repo

# dos-data-pipeline

## Project Purpose and Architecture Overview

This repo contains the **DOS (Data Operations Systems) data pipeline** — Scala/Spark ETL jobs and configuration for geo-location enrichment, segment aggregation, and political district bitmap preparation for Resonate's audience platform.

**Key pipelines:**
- **geo-location** — Daily enrichment of RCIDs with geographic + political district data (congress, state-senate, state-house); full backfill for historical correction. Feeds `ToBitmap` which sets marker bits.
- **segment-aggregator** — Aggregates behavioral segment data.

**Deployment:** GitHub Actions. Config in this repo deploys directly. Other pipelines are activated by `datapipeline-trigger` in `lambda-modeling`; their config comes from `services-api-configuration`.

---

## Repository Structure

```
src/main/scala/com/resonate/dos/
  GeoLocationDaily.scala             # Daily geo enrichment: IP → zip → district + district_source
  GeoLocationFullBackfill.scala      # Backfills historical dates (daily-on-multi-day-pixels approach)
  GeoLocationFull.scala              # Merges daily+backfill into rolling-full
  ToBitmap.scala                     # Converts geo-location output to bitmap; gates L2-confirmed bit
  DistrictResolver.scala             # Centralizes L2-vs-IP district coalescing + district_source derivation

src/test/scala/com/resonate/dos/
  GeoLocationDailyTest.scala
  GeoLocationFullBackfillTest.scala
  ToBitmapTest.scala
  ...

config/geo-location/
  zip-district-mappings/             # 4 CSV files (one per district type)
    zip-congress-mapping.csv          # 34,785 rows — L2 mode-per-zip, all 51 states
    zip-proposed-congress-mapping.csv # 8,362 rows — 2026 redistricted (CA/MO/NC/OH/TX/UT/VA)
    zip-state-senate-mapping.csv      # 33,527 rows
    zip-state-house-mapping.csv       # 32,620 rows
```

**Scala version:** 2.11.8 | **Spark version:** 2.4.3 (Spark 2 — not yet migrated to Spark 3)

---

## Key Commands

```bash
# Run all unit tests
sbt test

# Run a specific test suite
sbt "testOnly com.resonate.dos.GeoLocationDailyTest"
sbt "testOnly com.resonate.dos.ToBitmapTest"

# Build assembly JAR
sbt assembly

# Deploy — push to main triggers GitHub Actions deployment pipeline
```

---

## Key Concepts

### district_source Provenance (CDP-118946 / CDP-118947)

Every RCID in geo-location output carries a `district_source` column set by `DistrictResolver`:

| Value | Meaning |
|---|---|
| `L2_CONFIRMED` | L2 has a district AND `l2_party_confirmed = true`. **Only this value sets the L2-confirmed marker bit in ToBitmap.** |
| `L2_UNCONFIRMED` | L2 has a district AND `l2_party_confirmed != true`. |
| `IP_INFERRED` | No L2 district; NetAcuity IP → zip → 4-CSV-mapping found a district. |
| `null` | No district data from any source. |

**Observed distribution (dev 2026-05-12):** IP_INFERRED ~86.8%, L2_CONFIRMED ~9.9%, L2_UNCONFIRMED ~2.9%, null ~0.4%.

**ToBitmap gating (CDP-118947):** The L2-confirmed marker bit is set **only** when `district_source = "L2_CONFIRMED"`. Before this fix (CDP-118947) the bit was set unconditionally (broken for L2_UNCONFIRMED and IP_INFERRED rows).

### Zip→District Mappings (4-file Shape, CDP-118946)

The district lookup uses 4 separate CSV files (one per district type) rather than a single consolidated CSV. Schema: `zip, state, <district-type>`. Files version together as a snapshot.

Passed to Spark jobs as a single `zipDistrictMappingsBasePath` argument pointing at the directory. `DistrictResolver` reads all 4 files by convention from that path.

Deployed to S3 via terragrunt `configurations` entries in the geo-location pipeline. After updating CSVs, deploy via the geo-location GitHub Actions workflow.

**IP-inferred fallback flow:** `GeoLocationDaily` calls NetAcuity to resolve the RCID's IP to a zip, then joins the 4 CSVs for the district. State-leg mappings are many-to-many and collapsed to 1:1 using rank-based dedup with `sort_array` for deterministic tiebreaking.

### GeoLocationFullBackfill Rewrite (CDP-118946)

Rewritten as **daily-on-multi-day-pixels**: applies daily geo enrichment logic to each date in the backfill range rather than re-reading the prior rolling-full. Key changes:

- No longer takes `DistrictColumnList` arg — always re-derives all 4 districts (prevents stale labels)
- Reads existing `zip` from `fullDf`; applies same IP-fallback as Daily (no second NetAcuity call)
- `GeoLocationFull.expectedCols` extended with `zip` and `district_source`
- Orchestrator passes the date range as a rendered glob to backfill

### Backfill `Should Run Full` Bug Fix (CDP-118512)

A bug in the geo-location state machine (`geo_location.asl.json` in `step-function-workflow-orchestrator`) caused a pre-existing geo full for today to override backfill output. Fixed by adding an `IsPresent` check on `$.FullInputPath` before the path-equality choice. If backfill output is not propagating to the full-build stage, check the `Should Run Full` choice state.

---

## Project-Specific Rules and Gotchas

- **Spark 2.4.3**: This repo is still on Spark 2. Do not apply Spark 3-specific optimizations (EMR 7 migration for this repo is not yet underway).
- **State-leg mappings are many-to-many**: Do not assume 1:1 zip→district. `DistrictResolver` handles collapsing with `sort_array` for deterministic hash-based tiebreaking.
- **CSV versioning**: The 4 zip-district CSVs version together as a snapshot. Updating one requires coordinated update of all 4 if the underlying data changes.
- **`GeoLocationFull.expectedCols`**: If you add a new column to `GeoLocationDaily`, also add it to `expectedCols` in `GeoLocationFull.scala` so it flows through rolling-full and backfill.
- **NetAcuity in Backfill**: `GeoLocationFullBackfill` does NOT call NetAcuity. It reads the `zip` column already resolved by `GeoLocationDaily` and applies the CSV-based district lookup.
