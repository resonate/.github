# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **step-function-workflow-orchestrator** monorepo — the central deployment and orchestration hub for the X-Men data platform. It manages 50+ AWS Step Functions pipelines that run daily/weekly batch data workloads including identity graph construction, geo-location, audience segmentation, BEM delivery, and more.

## Repository Structure

```
├── pipelines/                       # One directory per pipeline
│   └── <pipeline-name>/
│       ├── config/                  # EMR configs (emr.json, params.json, steps.json)
│       │   ├── prod/
│       │   ├── dev/
│       │   ├── dev6/
│       │   └── integration/
│       ├── events/                  # Step Function input events
│       │   ├── prod/event.json
│       │   └── integration/event.json
│       ├── statemachine/            # ASL JSON state machine definition
│       └── tests/
│           └── integration/         # pytest integration test suite
│               ├── conftest.py
│               ├── create_golden_dataset_synthetic.py
│               └── test_<pipeline>.py
├── terraform/pipelines/             # Terragrunt configs (one per pipeline × env)
│   └── <pipeline-name>/
│       ├── dev/terragrunt.hcl
│       ├── dev6/terragrunt.hcl
│       ├── integration/terragrunt.hcl
│       └── prod/terragrunt.hcl
├── python_lambdas/                  # Shared Python Lambda functions
│   └── check-source-freshness/     # Config-driven S3/SSM freshness checker
└── .github/workflows/
    └── step_function.yml            # GitHub Actions deploy workflow
```

## Common Development Tasks

### Deploying a Pipeline

Use GitHub Actions (`X-Men Step Function Workflow`) to deploy any pipeline to any environment:

```bash
# Via GitHub CLI (always specify --ref when on a feature branch!)
gh workflow run step_function.yml \
  -f environment=dev \
  -f step_function=geo-location \
  --ref <your-branch-name>

# Check deployment status
gh run list --workflow=step_function.yml --limit=5
gh run watch <run-id>
```

**Environments:** `dev`, `dev2`–`dev9`, `integration`, `qa`, `prod`

**Available pipelines (selection):**
`behavior-stitch`, `damlam-preparation`, `experian-data-processing`, `geo-location`, `identity-graph`, `idsync-overlap`, `l2-processing`, `liveramp-stitch/thetradedesk`, `marketops-overlap`, `segment-adobe`, `segment-aggregator`, `segment-onboard`, `segment-transformer`, `sovrn-overlap`, `sovrn-weekly`, `tagav-prep`, `thirdparty-enrichment`, `total-sketch-first-party`, `vendor-stitch`

### Adding or Updating a Pipeline Config

Each pipeline's config in `pipelines/<pipeline-name>/config/<env>/` is deployed to S3 via Terragrunt. To add a new environment:

1. Copy the `dev/` config to `<env>/`, adjust paths/IAM roles.
2. Add a corresponding `terraform/pipelines/<pipeline-name>/<env>/terragrunt.hcl`.
3. Run the GHA `step_function.yml` workflow against the new env.

### Running Integration Tests

```bash
# Install test dependencies
pip install boto3 pytest

# Generate golden dataset
python3 pipelines/<pipeline-name>/tests/integration/create_golden_dataset_synthetic.py

# Run the tests (requires dev AWS credentials)
pytest pipelines/<pipeline-name>/tests/integration/ -v --timeout=3600
```

Integration tests follow this pattern:
1. Upload golden dataset to `resonate-core-datasets-dev`.
2. Start the `<pipeline-name>-integration` Step Function.
3. Poll until terminal state; assert SUCCEEDED or verify S3 output.

### Modifying the CheckSourceFreshness Lambda

The `python_lambdas/check-source-freshness/` Lambda supports multiple detection strategies:
- `timestamp_compare` — compares S3 LastModified timestamps
- `latest_directory_vs_ssm` — compares YYYYMMDD directory names against SSM last-run date
- `latest_file_prefix_vs_ssm` — lists files, extracts YYYYMM from names, compares to SSM

Adding a new source only requires a JSON config entry — no code change.

```bash
# Run Lambda unit tests
cd python_lambdas/check-source-freshness
pip install -r requirements.txt
python -m pytest tests/ -v
```

## Key Concepts

### EMR Versions

Pipelines are migrating from **EMR 5.34.0 → EMR 7.12.0** (CDP-118269). When touching a pipeline's `emr.json`:
- **EMR 5.x** uses `core-data-pipeline-latest.jar`
- **EMR 7.x / Spark 3** uses `core-data-pipeline-spark3-latest.jar`

### Integration Test Pattern

All integration tests follow the sovrn-overlap pattern:
- `conftest.py` provides session-scoped fixtures: SFN client, execution trigger, entered-states history
- Tests verify: step entered, S3 output exists, `_SUCCESS` marker present, schema correct
- InfluxDB metric steps are expected to fail in integration; tests accept SUCCEEDED or FAILED
- Auto-termination is set to 1800s idle timeout

### Terraform / Terragrunt

Each `terragrunt.hcl` includes:
- `schedule_expression` — set to `null` for integration/dev (no cron), cron expression for prod
- `replacement_vars` — substitutes template variables (ARNs, bucket names) in the ASL JSON
- `configurations` — maps S3 URIs for configs uploaded from the `pipelines/` directory

## Recent Changes (as of 2026-06)

- **CDP-118269**: Migrated `damlam-preparation` and `sovrn-weekly` from EMR 5.34.0 → 7.12.0 / Spark 3
- **CDP-118989 / Integration tests**: Added full integration test suites for `marketops-overlap`, `idsync-overlap`, `fusion-behavior-preprocess`, `behavior-stitch`, `segment-adobe`, `segment-onboard`, `segment-aggregator`, `liveramp-stitch/thetradedesk`, `total-sketch-first-party`
- **Removed**: `fusion-behavior-preprocess` pipeline (was dormant since 2023; caused unexpected prod run)
- **CDP-118890**: Added `experian-data-processing` pipeline with CheckSourceFreshness Lambda-based gating
- **CDP-118946**: Threaded zip→district CSV mappings through `geo-location` pipeline for IP-inferred district fallback
- **CDP-118512**: Fixed `geo-location` `Should Run Full` choice to preserve backfill output when today's full already exists

## Pipeline Lifecycle

1. Feature branch → PR → main auto-deploys to dev6 via GHA
2. Manual `workflow_dispatch` to `integration` → run integration tests
3. Manual `workflow_dispatch` to `prod` after review sign-off
