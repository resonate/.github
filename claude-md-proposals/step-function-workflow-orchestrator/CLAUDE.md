# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository is the **X-Men Step Function Workflow Orchestrator** — the central hub for all AWS Step Function–based data pipelines at Resonate. It contains pipeline ASL state machines, Python Lambdas, Terraform/Terragrunt infrastructure configs, and integration tests for pipelines across the CDP/data platform.

## Repository Structure

```
├── pipelines/                    # Per-pipeline directory
│   └── <pipeline-name>/
│       ├── statemachine/         # ASL JSON state machine
│       ├── config/               # EMR, params, steps configs (dev/prod/integration)
│       ├── events/               # Step Function input events (dev/prod/integration)
│       └── tests/
│           └── integration/      # pytest integration test suite
│               ├── conftest.py                          # Fixtures: SFN execution, S3 golden data
│               ├── create_golden_dataset_synthetic.py   # Synthetic S3 test data generator
│               └── test_<pipeline>.py                   # Smoke, content, schema, history tests
├── python_lambdas/               # Lambda functions (Python)
│   ├── check-source-freshness/   # Multi-strategy S3/SSM freshness gate
│   ├── dynamic-dates/            # Resolves <latest> / date-range tokens in event JSONs
│   ├── avmap-mapper/             # Maps rcid to avmap
│   └── ...
├── terraform/
│   └── pipelines/
│       └── <pipeline-name>/
│           └── <env>/terragrunt.hcl  # Per-env Terragrunt config
└── .github/workflows/
    └── step_function.yml             # Deploy workflow (environment + step_function inputs)
```

## Common Development Tasks

### Running Integration Tests

Integration tests run against live AWS Step Functions in the dev account. Each pipeline under `pipelines/<name>/tests/integration/` follows the same pattern:

```bash
# 1. Upload golden dataset (once per pipeline)
python3 pipelines/<name>/tests/integration/create_golden_dataset_synthetic.py

# 2. Run the tests
pytest pipelines/<name>/tests/integration/ -v --timeout=<N>
# Typical timeouts: 3600–7200 seconds depending on pipeline
```

Standard test classes (follow this pattern when adding new pipelines):
- `TestSmoke` — `SUCCEEDED` status, `_SUCCESS` marker in S3, non-empty output
- `TestContent` — schema validation, no-null checks, row count assertions
- `TestSchema` — column names and types
- `TestHistory` — Step Function execution history: expected states entered

### Deploying via GitHub Actions

**Always use `--ref <your-branch>` when deploying from a feature branch.** Without it, the workflow deploys from `main`.

```bash
# Deploy a Step Function to an environment
gh workflow run step_function.yml \
  --field environment=integration \
  --field step_function=<pipeline-name> \
  --ref feature/your-branch-name

# Check status
gh run list --workflow=step_function.yml --limit=5
```

**Environments:** `dev`, `dev2`–`dev7`, `integration`, `qa`, `prod`

### Python Lambda Development

Each Lambda lives in `python_lambdas/<name>/`. Tests use moto for S3/SSM mocking.

```bash
cd python_lambdas/<lambda-name>
python -m pytest tests/ -v
```

### Terraform / Terragrunt

Infrastructure configs at `terraform/pipelines/<name>/<env>/terragrunt.hcl`. Deploy via GitHub Actions — **do not run `terragrunt apply` locally against prod**.

## Pipeline Catalogue

| Pipeline | EMR Version | Key Jira |
|---|---|---|
| sovrn-weekly | EMR 7.12.0 (Spark 3) | CDP-118269 |
| damlam-preparation | EMR 7.12.0 (Spark 3) | CDP-118269 |
| experian-data-processing | EMR 7.x | CDP-118890 |
| geo-location | EMR 6.x | CDP-118946 |
| l2-processing | EMR 6.x | CDP-118512 |
| thirdparty-enrichment | EMR 6.x | — |
| total-sketch-first-party | EMR 6.15.0 | CDP-118989 |
| segment-aggregator | EMR 6.15.0 | CDP-118989 |
| marketops-overlap | EMR 6.x | CDP-118989 |
| idsync-overlap | EMR 6.x | CDP-118989 |
| tagav-prep | EMR 6.x | CDP-118989 |
| topic-tag-metrics | EMR 6.x | CDP-118967 |
| behavior-stitch | EMR 6.x | — |

## EMR Version Migration (CDP-118269)

Migrating pipelines from `emr-5.34.0` to `emr-7.12.0` (Spark 3). For each migrated pipeline:

1. Update `ReleaseLabel` in `config/emr.json` and `config/emr-on-demand.json`
2. Update the JAR filename in `config/prod/params.json`: `*-latest.jar` → `*-spark3-latest.jar`
3. Apply same changes to `config/integration/` files
4. Run the integration step function and confirm `SUCCEEDED`

## check-source-freshness Lambda

Supports multiple detection strategies (configured per pipeline in JSON):

| Strategy | Use Case |
|---|---|
| `timestamp_compare` | Compare S3 `LastModified` timestamps |
| `latest_directory_vs_ssm` | Latest YYYYMMDD subdirectory vs SSM last-run date |
| `latest_file_prefix_vs_ssm` | Latest date in flat filenames vs SSM (e.g. Experian `YYYYMM_*.gz`) |

When `has_new_data: false`, the pipeline short-circuits without launching an EMR cluster.

## Integration Test Pattern

Each new pipeline integration test suite MUST include:

1. **`config/integration/emr.json`** — small on-demand cluster, `AutoTerminationPolicy.IdleTimeout: 1800`
2. **`config/integration/params.json`** — Test IAM roles (`CoreDataPipelineEC2ResourceRoleTest` / `CoreDataPipelineRoleTest`), `Environment: INTEGRATION`, fixed output date `20260101`
3. **`events/integration/event.json`** — `ExpiryTimeout` sized to pipeline (e.g. 3600–7200), empty preconditions
4. **`terraform/pipelines/<name>/integration/terragrunt.hcl`** — `schedule_expression = null`
5. **`tests/__init__.py`** and **`tests/integration/__init__.py`** — package markers for pytest discovery
6. **`create_golden_dataset_synthetic.py`** — generates synthetic parquet input, uploads to `resonate-core-datasets-dev`
7. **`conftest.py`** — `autouse=True` golden-data upload, SFN execution fixture with timeout, execution history fixture
8. **`test_<name>.py`** — standard smoke/content/schema/history test classes

## Dynamic Dates Lambda

Resolves tokens in event JSON `configurations` values:

- `<latest>` — last date-named S3 directory with `_SUCCESS` marker
- `<latest@refName, %Y%m%d>` — flat-file mode: last date embedded in filename, gated by a sentinel file suffix
- `<@refName>` — reuse a captured date from a previous `<latest@refName,...>` field
- `<N, unit>` / `<latest, %Y%m%d>` — date arithmetic

## QA Environments

`qa` state machines (e.g. `geo-location-qa`) share source data with prod but write to `*-qa` buckets. Cron is disabled — manual trigger only. Deploy via `step_function.yml` with `environment: qa`.

## Key Constraints

- **Never force-push to `main`**
- **Never run `terragrunt apply` against prod locally**
- **Specify `--ref` when triggering GHA workflows from a feature branch**
- Branch naming: `feature/{JIRA-ID}-{kebab-slug}` (e.g. `feature/CDP-118269-sovrn-weekly-emr7`)
- Fusion-behavior-preprocess pipeline was removed (PR #734) — do not re-add it
