# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains **Batch Audience Delivery — Syndication** workflows: the Lambda functions and Step Function state machines that publish formatted audience data files to syndicated delivery vendors (OpenX, Experian, Viant/OpenX). It works as a consumer of output from `batch-expression-modeling`'s formatter step.

### Supported Vendors

| Vendor | Lambda | State Machine |
|--------|--------|--------------|
| OpenX | `openx-publish-data-files` | `batch-audience-delivery-syndication` |
| Experian | `experian-publish-data-files` | `experian-syndication-workflow` |
| Viant | `viant-publish-files` | (see viant workflow) |
| BlockGraph | `blockgraph-create-taxonomy-file` | BlockGraph delivery workflow |

## Repository Structure

```
├── workflows/
│   ├── lambdas/
│   │   ├── openx-publish-data-files/       # Renames + copies CSV.gz files to OpenX S3
│   │   ├── blockgraph-create-taxonomy-file/ # Generates BlockGraph metadata CSV(s)
│   │   └── ...
│   └── step-functions/                     # ASL state machine definitions
├── terraform/
│   └── workflows/lambdas/                  # Terraform per-lambda per-env
└── .github/workflows/                      # CI/CD
```

## Common Development Tasks

### Python Lambda Development

Each Lambda has its own directory under `workflows/lambdas/<name>/`:

```bash
# Run tests
cd workflows/lambdas/<lambda-name>
pytest test/ -v

# Example
cd workflows/lambdas/openx-publish-data-files
pytest test/ -v  # 6 tests
```

### Deploying Lambdas

Lambda deployment is via GitHub Actions (see `.github/workflows/`). Deploy to a specific environment:

```bash
gh workflow run <workflow>.yml \
  -f environment=dev \
  --ref <your-branch>
```

### Terraform / Terragrunt

```bash
cd terraform/workflows/lambdas/<lambda-name>/<env>
aws sso login
terragrunt plan
terragrunt apply
```

## Key Lambda: openx-publish-data-files

Copies Spark-formatted `.csv.gz` files from S3 source to OpenX destination, renaming to the pattern `resonate_syndication_{date}_Data_{N}.csv.gz`.

**Important:** The extension is hardcoded as `.csv.gz` — do NOT revert to dynamic `split('.', 1)` extension parsing. This was fixed in CDP-118955 because the upstream `batch-expression-modeling` formatter (CDP-118857) changed Spark's codec-suffix separator from `-c000.csv.gz` to `.c000.csv.gz`, which broke dynamic parsing and silently stopped all OpenX audience refreshes.

**Part number extraction:** Uses regex `part-(\d+)-.*\.csv` to find the part number, then increments by 1 for 1-indexed output filenames.

## Key Lambda: blockgraph-create-taxonomy-file

Generates BlockGraph metadata CSVs for a delivery run (CDP-118915):

- **Initial format** (13 fields) — net-new segments not yet in BlockGraph
- **Refresh format** (8 fields) — segments already known to BlockGraph

**Resolution:**
- Syndicated mode: queries BlockGraph ADS group to resolve audience set
- Custom mode: uses `audience_key_list` from event, filters to BlockGraph group hierarchy

**SPI field:** Constant `N` — we deliver BGIDs (not SPI source data), so the "Created using Sensitive Personal Information" flag is `N` taxonomy-wide (Q5 resolved, CDP-118915). Env var `spi_value` can override if needed.

**Output path:** `<prefix>/batch-delivery-payload/metadata/resonate_metadata_{initial,refresh}_{ts}.csv`

## Key Concepts

### Source Path Layout (Post CDP-118857)

The `batch-expression-modeling` formatter outputs data partitioned as:
```
<prefix>/date=<YYYYMMDD>/vendor=<vendor>/method=<method>/akey=<key>/part-*.csv.gz
```

The source_prefix for syndication lambdas must concatenate `vendor=*/method=av/` (NOT the legacy `method=av/vendor=*/`). This was fixed in CDP-118937 for Experian, Yahoo, and custom delivery.

### File Extension Warning

Spark's `.partitionBy()` changes the codec suffix separator from `-c000.csv.gz` to `.c000.csv.gz`. Any lambda that dynamically parses file extensions from Spark output filenames will break after a `partitionBy` change. Always hardcode the expected extension (`.csv.gz`) after confirming `list_csv_files` already filters to that extension.

### Delivery State Marker

`blockgraph-create-taxonomy-file` reads the Delivery State marker from `state/known-segments/` to identify which PSIDs have already been delivered (refresh vs. initial routing).

## Recent Changes (May–June 2026)

- **openx-publish-data-files** (PR #46, CDP-118955): Hardcoded `.csv.gz` extension — fixes broken OpenX syndication after CDP-118857 Spark `partitionBy` change caused malformed filenames (last good delivery was 2026-04-26)
- **blockgraph-create-taxonomy-file** (PR #48, CDP-118915): New Lambda implementing BlockGraph taxonomy file generation — 18 unit tests, supports both syndicated and custom audience modes, SPI=N hardcoded
- **source_prefix path order** (PR #42, CDP-118937): Swapped `method=av/vendor=*` → `vendor=*/method=av/` to match new formatter output layout
