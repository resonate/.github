# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **X-Men Step Function Workflow Orchestrator** — AWS Step Functions state machines and supporting infrastructure that orchestrate Resonate's core data pipelines. Each pipeline is a self-contained directory under `pipelines/` with its own EMR cluster config, Step Function ASL definition, Terraform infrastructure, and integration tests.

### Active Pipelines

| Pipeline | Purpose |
|----------|---------|
| `behavior-stitch` | Stitches behavioral signal to RCIDs |
| `topic-aggregation` | Aggregates topic signals |
| `idsync-overlap` | Computes identity-sync overlap metrics |
| `damlam-preparation` | Prepares DAMLAM data for audience modeling |
| `marketops-overlap` | Computes market-ops overlap metrics |
| `sovrn-weekly` | Weekly Sovrn behavioral preprocess |
| `geo-location` | GeoLocation daily + full backfill pipeline |
| `l2-processing` | L2 voter data processing and stitch |
| `thirdparty-enrichment` | Third-party enrichment (TPE) |
| `experian-data-processing` | Experian offline graph pipeline |
| `total-sketch-first-party` | First-party total sketch |
| `state-legislative-districts-aggregation` | State legislative district bitmap aggregation |

## Repository Structure

```
├── pipelines/<pipeline-name>/
│   ├── config/
│   │   ├── emr.json                # EMR cluster config (prod)
│   │   ├── emr-on-demand.json      # EMR on-demand cluster config
│   │   ├── prod/params.json        # Pipeline parameters (prod)
│   │   ├── dev/params.json         # Pipeline parameters (dev)
│   │   ├── integration/            # Integration test configs
│   │   └── qa/                     # QA environment configs
│   ├── statemachine/*.asl.json     # Step Function ASL definition
│   ├── events/<env>/event.json     # EventBridge event payload
│   ├── tests/integration/          # Python integration test suite
│   └── terraform/                  # Terraform/Terragrunt modules
├── terraform/pipelines/<pipeline>/<env>/terragrunt.hcl
├── .github/workflows/              # GitHub Actions
└── lambdas/
    └── dynamic-dates/              # Lambda for resolving dynamic date paths
```

## Common Development Tasks

### Deploying a Pipeline

```bash
# Deploy via GitHub Actions workflow_dispatch
# Navigate to Actions → "X-Men Step Function Workflow"
# Select: environment (dev/qa/integration/prod), step_function name

# Or via gh CLI:
gh workflow run step_function.yml \
  -f environment=dev \
  -f step_function=<pipeline-name>
```

### Running Integration Tests

```bash
# Upload golden data (once per pipeline)
python3 pipelines/<pipeline>/tests/integration/create_golden_dataset_synthetic.py

# Run tests (allow up to 90 min for EMR)
pytest pipelines/<pipeline>/tests/integration/ -v --timeout=6000
```

Integration tests:
- Deploy integration-specific Step Function via terragrunt
- Upload synthetic golden data to `s3://resonate-core-datasets-dev/integration-test/<pipeline>/`
- Execute the `<pipeline>-integration` state machine
- Assert S3 output, `_SUCCESS` markers, and execution history

### Terraform / Terragrunt

```bash
# Plan infrastructure changes
cd terraform/pipelines/<pipeline>/<env>
terragrunt plan

# Apply (requires AWS SSO login)
aws sso login
terragrunt apply
```

### EMR Configuration

Each pipeline's `config/emr.json` specifies:
- `ReleaseLabel` — EMR version (currently migrating to `emr-7.12.0`)
- Instance types and counts
- YARN memory/vcore settings
- `AutoTerminationPolicy.IdleTimeout`

**EMR 5 → 7 migration notes (CDP-118269):**
- Bump `ReleaseLabel` from `emr-5.34.0` → `emr-7.12.0`
- Update JAR path to `core-data-pipeline-spark3-latest.jar`
- Fix YARN vcores: set `yarn.nodemanager.resource.cpu-vcores` explicitly (EMR 7 no longer auto-detects with multiplier)
- Reduce executor memory for integration: r5.xlarge YARN limit is ~24576 MB with higher overhead factor on EMR 7

### Dynamic Dates Lambda

The `lambdas/dynamic-dates/` Lambda resolves `<latest>` and `<latest@ref,%Y%m%d>` tokens in event JSON path fields.

**Two resolution modes:**
- **Directory mode** (prefix ends with `/`): lists subdirectories, picks latest with `_SUCCESS`
- **Flat-file mode** (prefix does NOT end with `/`): lists objects, parses date from key. The text after `>` is a **completion sentinel** — a date is only valid if `prefix + DATE + sentinel_suffix` is an actual S3 key

```json
// Example dual-field flat-file pattern (TapAd Digital Graph)
"TapAdDigitalGraphSuccessMarker": "s3://bucket/path/resonate_ids_full_<latest@tapadDate,%Y%m%d>_000000.parquet.master.trigger",
"TapAdDigitalGraphData":          "s3://bucket/path/resonate_ids_full_<@tapadDate>_000000_part-*.parquet"
```

## Key Concepts

### Pipeline Anatomy

Each pipeline Step Function follows a standard pattern:
1. **EventBridge trigger** → Lambda (`dynamic-dates`) resolves date tokens
2. **Precondition check** → Lambda (`precondition-checker`) validates S3 inputs
3. **EMR Create Cluster** → Provisions cluster
4. **EMR Add Step(s)** → Runs Spark job(s)
5. **EMR Terminate Cluster** → Cleans up
6. **SNS Notify** → Success or failure notification

### Environments

| Environment | Account | Purpose |
|-------------|---------|---------|
| `dev` / `dev2`-`dev7` | dev | Individual developer sandboxes |
| `integration` | dev | CI integration testing (manual trigger, no cron) |
| `qa` | dev | QA testing (manual trigger, no cron) |
| `nonprod` | nonprod | Pre-production |
| `prod` | prod | Production (daily EventBridge cron) |

### JAR Naming Convention

- `core-data-pipeline-latest.jar` → Spark 2 (EMR 5) build (legacy)
- `core-data-pipeline-spark3-latest.jar` → Spark 3 (EMR 7) build (current standard)

Pipeline-specific JARs (e.g., `identity-graph-latest.jar`) live in `s3://resonate-core-applications/`.

### Geo-Location Pipeline

The `geo-location` pipeline runs `GeoLocationDaily` and `GeoLocationFullBackfill` with:
- **zip→district CSV mappings** (4 files): `zip-congress-mapping.csv`, `zip-proposed-congress-mapping.csv`, `zip-state-senate-mapping.csv`, `zip-state-house-mapping.csv`
- Deployed to `s3://resonate-core-applications{-env}/dos/geo-location/configs/zip-district-mappings/`
- `district_source` provenance: `L2_CONFIRMED`, `L2_UNCONFIRMED`, `IP_INFERRED`

**State machine backfill logic:**
The `Should Run Full` choice state checks `$.FullInputPath` before path-equality comparison — when backfill ran, `Use Backfill Output` sets `$.FullInputPath` to the backfill path and that must be preferred (CDP-118512 fix).

### L2 Processing Pipeline

Wired to discover TapAd Digital Graph data via flat-file sentinel pattern and LiveRamp Tradedesk Agg via directory pattern. Parameters `TapAdDigitalGraphSuccessMarker` / `TapAdDigitalGraphData` / `TapAdDigitalGraphLookbackDays` control the new HEM→RCID derivation path.

## Recent Changes (May–June 2026)

- **EMR 7 migration** (CDP-118269): `behavior-stitch`, `topic-aggregation`, `idsync-overlap`, `damlam-preparation`, `marketops-overlap`, `sovrn-weekly` all migrated to EMR 7.12.0 / Spark 3
- **Integration test suites** added for `marketops-overlap`, `idsync-overlap`, `fusion-behavior-preprocess`, `behavior-stitch`
- **QA environments** added for `geo-location`, `l2-processing`, `thirdparty-enrichment`, `total-sketch-first-party`
- **`fusion-behavior-preprocess` removed** (CDP incident #16828 — pipeline was dormant; deploy on 2026-05-28 caused EC2 capacity failure)
- **geo-location zip→district mappings** rewritten to L2 canonical namespace for NH, MA, MN, VT, MD, SD, ND state house and senate (CDP-118946)
- **L2 stitch** wired to TapAd Digital Graph for HEM→RCID via flat-file sentinel pattern (CDP-118512)
- **Geo backfill fix**: `Should Run Full` choice now uses `IsPresent` check on `$.FullInputPath` before path-equality (CDP-118512)
- **Experian pipeline**: switched source from cross-account `tapad-resonate` to `resonate-experian-rld` bucket (CDP-118890)
