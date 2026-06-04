# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**batch-audience-delivery-syndication** manages batch audience syndication workflows — Lambda functions and Step Function state machines that deliver audience data to third-party advertising platforms (BlockGraph/FreeWheel, OpenX, Viant). Each partner has its own subdirectory of Lambdas and a corresponding Step Function workflow.

**Tech stack:** Python 3 Lambdas, AWS Step Functions (ASL JSON), Terraform/Terragrunt for infrastructure.

## Repository Structure

```
lambdas/
  blockgraph-create-taxonomy-file/  # Creates BlockGraph taxonomy CSV
  blockgraph-rename-files/          # Renames BlockGraph output part files
  blockgraph-publish-files/         # Uploads files to BlockGraph SFTP/S3
  openx-get-metrics/                # Retrieves OpenX delivery metrics
  openx-publish-data-files/         # Publishes audience data to OpenX
  openx-publish-metrics/            # Publishes metrics for OpenX
  viant-create-taxonomy/            # Creates Viant taxonomy mapping
  viant-publish-audience-data/      # Delivers audience data to Viant
  viant-publish-metrics/            # Publishes Viant delivery metrics
step-functions/
  blockgraph-*/                     # BlockGraph syndication state machines
  openx-*/                          # OpenX syndication state machines
  viant-*/                          # Viant syndication state machines
terraform/
  lambdas/<name>/<env>/             # Terragrunt config per Lambda per env
  step-functions/<name>/<env>/      # Terragrunt config per SF per env
```

## Common Development Tasks

### Running Lambda Tests

Each Lambda has its own `tests/` directory. Run from the Lambda's root:

```bash
cd lambdas/<lambda-name>
pip install -r requirements.txt
python -m pytest tests/
```

### AWS Authentication

```bash
aws sso login
```

Required before any local AWS interactions or uploading to S3.

### Deploying via GitHub Actions

All deployments go through GitHub Actions — do not run `terragrunt apply` locally.

Workflows are triggered by pushes to `lambdas/**` or `step-functions/**`, or manually via `workflow_dispatch`.

## Key Concepts

### Partners / Vendors

- **BlockGraph (FreeWheel):** Audience data delivered as CSV files uploaded to BlockGraph's SFTP endpoint or S3. The workflow creates a taxonomy file, renames/prepares the part files, then publishes. SSM Parameter Store holds the SFTP credentials.
- **OpenX:** Audience data published via OpenX API. The `openx-publish-data-files` Lambda appends `.csv.gz` extension to output files. An API key expiry check is built into `openx-get-metrics`.
- **Viant:** Full workflow added in Jan 2026 (CDP-118042). Taxonomy creation → audience data publishing → syndicated delivery via `viant-syndicated` Step Function.

### BlockGraph Lambdas (added Jun 2026, CDP-118915/16/17)

Three new Lambdas implement the BlockGraph file delivery pipeline:

1. **`blockgraph-create-taxonomy-file`** — Generates a CSV taxonomy file with `SPI=N` (product-confirmed non-sensitive). Does not verify `ExpectedBucketOwner` (intentional).
2. **`blockgraph-rename-files`** — Renames `part-N.csv.gz` output files to the naming convention required by BlockGraph. Regex matches bare `part-N.csv.gz` filenames.
3. **`blockgraph-publish-files`** — Uploads renamed files to BlockGraph destination. Uses SSM boto config; upload ordering is deterministic. IAM permissions are least-privilege.

### Source Prefix Convention

The `source_prefix` field in syndication configs must match the formatter's output path layout (CDP-118937). If the upstream formatter changes its path structure, update `source_prefix` in the corresponding syndication Lambda config.

## Project-Specific Rules and Gotchas

- **Never run `terragrunt apply` locally** — all infra changes must go through GitHub Actions.
- **SPI=N is intentional** in BlockGraph taxonomy: the product team confirmed this value; do not change it.
- **`ExpectedBucketOwner` is intentionally absent** in `blockgraph-create-taxonomy-file` — do not add it.
- **`.csv.gz` extension** is hardcoded in `openx-publish-data-files` (CDP-118955) — the extension is always `.csv.gz` regardless of upstream output.
- **PagerDuty alerting** is wired up for OpenX and Viant Step Functions (CDP-118463). Check `step-functions/<workflow>/prod/` for the SNS topic ARN configuration.
- **Viant syndicated workflow** (`viant-syndicated`) is separate from the individual Viant Lambdas — it is the entry-point state machine that orchestrates the full Viant delivery sequence.
