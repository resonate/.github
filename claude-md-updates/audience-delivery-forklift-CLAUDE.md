# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**audience-delivery-forklift** is an Apache Spark pipeline (Scala 2.11 / Spark 2.4.8 / Java 8 / SBT) that processes audience data for delivery. It handles three stages: **modeling**, **identity stitching**, and **vendor-specific formatting** for 23+ advertising platforms. Actual delivery to vendors is handled downstream.

> **Deprecation status:** This repo is in the process of being deprecated. Components still in use:
> - **DAM modeling + stitch** — Active, minimal support only.
> - **Legacy LAM modeling + stitch** — Active, likely to be deprecated soon.
> - **AudienceDeliver (formatting/predelivery)** — Active, planned to move to the batch system long term.
>
> All other components (CookieJarEvaluator / AV/Retargeting) are already deprecated.

## Repository Structure

```
src/main/scala/
  apps/
    modeling/         # CookieJarEvaluator (deprecated AV/Retargeting)
    model/
      lam/            # LookAlikeModelingDriver (Legacy LAM)
      dam/            # AudiencePrediction (DAM)
    stitch/           # Stitch — RCID → vendor identity key
    deliver/          # AudienceDeliver + per-vendor formatters
docs/
  architecture.md     # System architecture overview (added Jun 2026)
build.sbt
```

## Pipeline Stages

```
SuperTagAv / SuperBehavior / SuperQueue
    ↓
Modeling (CookieJarEvaluator / Legacy LAM / DAM)
    ↓
MatchedId (rcid + score + identity columns)
    ↓
Stitch (identity resolution — DAM + Legacy LAM only)
    ↓
vkey (vendor-specific identity)
    ↓
AudienceDeliver (formatting / pre-delivery)
    ↓
Vendor-specific output
```

## Common Development Tasks

### Build and Test

```bash
# Compile
sbt compile

# Run all tests
sbt test

# Run a specific test class
sbt "testOnly apps.deliver.SomeDeliverTest"

# Package JAR for EMR
sbt assembly
# Output: target/audience-delivery-forklift-assembly-*.jar
```

### AWS Authentication

```bash
aws sso login
```

### Deployment

Deployments are managed via GitHub Actions (GitOps). See the [deployment docs](https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/3683057840/Pipeline+Scala+SBT+GitOps).

Production JAR path: `s3://resonate-core-applications/audience-delivery-forklift/jars/audience-delivery-forklift-latest.jar`

## Key Concepts

### Entry Points

| Application | Class | Purpose |
|---|---|---|
| CookieJarEvaluator | `apps.modeling.CookieJarEvaluator` | Evaluates boolean expressions against attribute bitmaps |
| LookAlikeModeling | `apps.model.lam.LookAlikeModelingDriver` | Generates look-alike audiences from seed behavior |
| AudiencePrediction (DAM) | `apps.model.dam.AudiencePrediction` | SVM-based audience membership prediction |
| Stitch | `apps.stitch.Stitch` | Maps RCIDs to vendor-specific identity keys |
| AudienceDeliver | `apps.deliver.AudienceDeliver` | Formats and writes vendor delivery files |

### Delta Delivery (Meta Pre-delivery)

When a `previousInputPath` is provided, AudienceDeliver computes added/removed records and tags them with `A`/`R` status. The Meta pre-delivery implementation uses a **full outer join** (not `except`/`intersect`) to compute the delta — this preserves a `keep` partition for records that remain unchanged, which is required for Meta's delivery format (CDP-118005, merged Mar 2026).

### Identity Stitching

Supported stitch types: `tdid_array`, `rawrcid_array`, `ip`, `hemsha2`, `viant_id`, `tmid`, `mmid`, `tapad_deviceid`. Uses **quantile-based limiting** (not Spark `.limit()`) to efficiently cap audience size.

### Exclusion Expression List (CDP-115041/115754)

The RCID exclusion path was replaced with an exclusion expression list. When configuring exclusions, pass a list of expressions rather than an S3 path to RCIDs.

### Vendor Support

The `akey` column was removed from parquet output files (CDP-115105, Jan 2025). Do not re-add it. Supported vendors include: Meta, TikTok, Liveramp, The Trade Desk, Roku, Yahoo, Adobe, Bluekai, Dstillery, and others. TikTok activation was added in Mar 2025 (CDP-115911). LiveRamp PMP is treated as a distinct vendor from standard LiveRamp (CDP-114282).

## Project-Specific Rules and Gotchas

- **Repo is in deprecation mode** — prefer minimal changes; new audience delivery work should go to `batch-expression-modeling`.
- **DAM + Legacy LAM stitch** is still active; do not remove the Stitch stage.
- **AudienceDeliver (pre-delivery/formatting)** is still active; it will eventually move to the batch system.
- **Do not use `.limit()`** to cap audience size — use quantile-based limiting instead to avoid expensive shuffles.
- **Delta delivery uses full outer join**, not set operations — see `AudienceDeliver` implementation introduced in CDP-118005.
- **akey column is intentionally absent** from parquet output — do not add it back (removed in CDP-115105).
