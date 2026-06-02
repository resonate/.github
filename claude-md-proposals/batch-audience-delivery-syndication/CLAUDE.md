# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **Batch Audience Delivery Syndication** workflows — Python Lambda functions and AWS Step Function state machines that handle syndicated delivery of audience data to external partners (OpenX and Viant).

## Repository Structure

```
├── workflows/
│   ├── lambdas/
│   │   ├── openx-publish-data-files/   # Renames + delivers CSV files to OpenX S3 bucket
│   │   │   ├── openx_publish_data_files.py
│   │   │   └── test/
│   │   ├── viant-publish-files/         # Similar delivery for Viant
│   │   └── ...
│   └── step-functions/
│       ├── openx-syndication-workflow/  # ASL JSON for OpenX delivery state machine
│       └── viant-syndication-workflow/  # ASL JSON for Viant delivery state machine
├── terraform/                           # Terragrunt infrastructure configs
└── .github/workflows/                   # GitHub Actions deployment workflows
```

## Common Development Tasks

### Running Lambda Tests

```bash
# openx-publish-data-files tests
cd workflows/lambdas/openx-publish-data-files
python -m pytest test/ -v

# viant-publish-files tests
cd workflows/lambdas/viant-publish-files
python -m pytest test/ -v
```

### Deploying

Lambda and Step Function deployments are managed via GitHub Actions. Always specify `--ref` when deploying from a feature branch:

```bash
# Deploy Lambdas
gh workflow run <lambda-workflow>.yml -f environment=dev --ref <your-branch>

# Deploy Step Functions
gh workflow run <step-function-workflow>.yml -f environment=dev --ref <your-branch>
```

## Key Concepts

### openx-publish-data-files Lambda

Lists CSV files from S3 (filtered to `.csv.gz`), renames them to the expected OpenX naming convention (`AM_ResonateDataAlliance_<n>_<date>_Data_<N>.csv.gz`), and copies to the OpenX destination bucket.

**File extension handling (post-CDP-118857):**
- Spark's `partitionBy("method")` changed the codec suffix separator from `-c000.csv.gz` to `.c000.csv.gz`
- The extension is now **hardcoded** to `.csv.gz` (not parsed from the source filename)
- `PART_PATTERN` matches both `-c000` and `.c000` separators for part number extraction
- Do NOT revert to dynamic extension parsing via `split('.', 1)` — this broke OpenX delivery (CDP-118955)

### Source Prefix Path Layout (post-CDP-118857)

The batch-expression-modeling formatter flipped its output partition order:
- **Old**: `<prefix>/date=*/method=av/vendor=*/akey=*/`
- **New**: `<prefix>/date=*/vendor=*/method=*/akey=*/`

The `source_prefix` concatenation in both syndication workflows was updated to match the new layout (CDP-118937). `method=av` remains hardcoded since these state machines handle AV-only delivery.

### PagerDuty Alerting (CDP-118463)

Both `openx-syndication-workflow` and `viant-syndication-workflow` send PagerDuty alerts on failure. The EventBridge rule uses a single `DetailType` for compatibility. Do not add multiple `DetailType` values.

### Delivery Flow

```
formatter output (S3) → list CSV files → rename → copy to partner bucket
```

The formatter output uses the new path layout (see above). The `list_csv_files` step already filters to `.csv.gz`, so extension logic downstream only needs to handle renaming, not filtering.

## Recent Changes (as of 2026-06)

- **CDP-118955 (PR #46)**: Hardcoded `.csv.gz` extension in `openx-publish-data-files` — fixed silent delivery failure after Spark `partitionBy` changed codec suffix separator
- **CDP-118937 (PR #42)**: Swapped `source_prefix` concatenation order to match new formatter path layout (`vendor=*/method=*/` instead of `method=av/vendor=*/`)
- **CDP-118463 (PR #40)**: Added PagerDuty failure alerting to OpenX and Viant syndication workflows

## Testing Guidelines

- Parametrize tests for **both** the old (`-c000.csv.gz`) and new (`.c000.csv.gz`) source filename suffixes — the `PART_PATTERN` regex must handle both
- When adding new syndication partners, follow the `openx-publish-data-files` pattern: hardcode the output extension, use the new vendor/method partition order
- Ensure PagerDuty `DetailType` uses a single string value in EventBridge rules
