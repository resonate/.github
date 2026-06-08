# batch-audience-delivery-syndication

This repository contains the **vendor syndication Lambda functions** for Resonate's batch audience delivery platform. Each Lambda handles a step in the pipeline that takes stitched audience data from S3 and delivers it to third-party advertising platforms (OpenX, Viant, BlockGraph/FreeWheel, Experian, Yahoo, etc.).

## Repository Structure

```
workflows/
  lambdas/                     # One subdirectory per syndication Lambda
    openx-publish-data-files/  # Upload to OpenX S3 bucket
    viant-publish-files/       # Upload to Viant S3 bucket
    experian-publish-files/    # Upload to Experian S3 bucket
    yahoo-publish-files/       # Upload to Yahoo S3 bucket
    blockgraph-create-taxonomy-file/  # T06: generate BlockGraph metadata CSV
    blockgraph-rename-files/          # T07: concatenate + rename Spark output files
    blockgraph-publish-files/         # T08: upload files to BlockGraph S3 bucket
    ...
  step-functions/              # ASL Step Function definitions for each vendor

terraform/
  workflows/
    lambdas/<lambda-name>/
      terragrunt.hcl           # Lambda Terragrunt root config
      dev/                     # Dev IAM roles / env config
      prod/                    # Prod IAM roles / env config

.github/
  workflows/
    lambda.yml                 # Deploy Lambdas (manual dispatch dropdown)
    all.yml                    # CI: auto-discovers all lambdas for testing
```

## Lambda Development

Each Lambda has its own directory under `workflows/lambdas/<name>/` with:
- `lambda_function.py` — Lambda handler
- `requirements.txt` — Python dependencies
- `.python-version` — Python version pin
- `tests/` — Unit tests
- `sonar-project.properties` — SonarCloud configuration

**Run tests for a Lambda:**
```bash
cd workflows/lambdas/<lambda-name>
pip install -r requirements.txt
python -m pytest tests/ -v
```

**Deploy a Lambda:**
```bash
# Via GitHub Actions workflow dispatch (preferred)
gh workflow run lambda.yml -f lambda=<lambda-name> -f environment=dev --ref <your-branch>
```

## Common Lambda Pattern

Syndication Lambdas follow a consistent pattern inherited from `openx-publish-data-files` and `viant-publish-files`:

**Event shape:**
```json
{
  "source_bucket": "resonate-audience-delivery",
  "source_prefix": "path/to/spark/output/",
  "output_bucket": "resonate-audience-delivery",
  "output_prefix": "path/to/output/",
  "date_started": "2026-05-01T00:00:00Z"
}
```

**Execution pattern:**
- Audiences processed concurrently via thread pool
- Per-audience failures propagate (no silent swallows)
- `dry_run` mode skips actual uploads (used in testing)
- State files written to `state/known-segments/run_date=YYYYMMDD/` after successful delivery

## S3 Source Path Layout

Spark formatter output uses the partition order: `<prefix>/date=*/vendor=*/method=*/akey=*/`

**Important:** A path layout change landed with CDP-118857/CDP-118937 — the formatter swapped partition order from `method=av/vendor=...` to `vendor=.../method=.../`. All ASL Step Function state machines concatenating `source_prefix` must use the new order. Do NOT use the old `method=av/vendor=` order.

## BlockGraph / FreeWheel Syndication Pipeline

BlockGraph is a new vendor (CDP-118694 epic) using a distinct 3-Lambda chain (T06 → T07 → T08):

### T06: `blockgraph-create-taxonomy-file`
Creates the metadata CSV (`taxonomy.csv`) required by BlockGraph, listing all audience segments with their IDs and names. Output goes to `<output_prefix>/metadata/`.

### T07: `blockgraph-rename-files` (CDP-118916)
Takes per-audience Spark output (`akey=<audience_key>/part-*.csv.gz`) and produces a single renamed file per audience following BlockGraph's convention: `resonate_<audience_key>_<ts>.csv.gz`.

**Concatenation strategy (cheapest valid path):**
- **0 parts** → valid empty gzip
- **1 part** → server-side `CopyObject` rename
- **multi-part, all but last ≥5 MiB** → server-side `UploadPartCopy` (bytes stay in S3)
- **multi-part with small parts** → download-concat-upload fallback through Lambda

Concatenated gzip streams are valid per RFC 1952 (byte-level concatenation, no re-encoding).

**Paths:**
- Read: `<source_prefix>/akey=<audience_key>/part-*.csv.gz`
- Write: `<output_prefix>/resonate_<audience_key>_<ts>.csv.gz` (timestamp from `date_started`, colons → underscores)

### T08: `blockgraph-publish-files` (CDP-118917)
Uploads the renamed segment files (T07) and metadata CSVs (T06) to BlockGraph's S3 bucket using BlockGraph-issued cross-account credentials, then writes the delivery-state delta (net-new PSIDs) to our own bucket.

**Cross-account upload pattern:**
- Download with our execution-role client → stream through temp file → upload with BG-credentialed client
- BG credentials come from SSM: `/resonate/cdp-118203/blockgraph/aws-access-key-id` and `/resonate/cdp-118203/blockgraph/aws-secret-access-key` (env-overridable via `BG_ACCESS_KEY_PARAM` / `BG_SECRET_KEY_PARAM`)
- Lambda IAM role has **no** permissions on BG's bucket — all writes go through SSM keys

**BG upload destinations:**
- Metadata CSV → `auto/segment/metadata/`
- Segment files → `auto/segment/upload/`

**State delta:**
- Reads known-PSID set from `state/known-segments/`
- Unions PSIDs from this run's metadata files (auto-detects `Participant Segment ID` column for both 13- and 8-field metadata layouts)
- Writes `delta = run − known` to `…/state/known-segments/run_date=YYYYMMDD/run_<ts>.csv`
- Empty delta → nothing written; failure mid-upload → state file not written

## Terraform / Deployment

Each Lambda has per-environment Terraform config in `terraform/workflows/lambdas/<name>/dev/` and `prod/`. IAM roles follow the principle of least privilege — BG Lambdas have no cross-account bucket permissions; only the upload Lambda has SSM access to BG credentials.

**Add a new Lambda to the deploy dropdown:**
Edit `.github/workflows/lambda.yml` to add the Lambda name to the `workflow_dispatch` choices. The `all.yml` CI workflow auto-discovers Lambda directories.

## Rules and Gotchas

- **SSM credential param names for BlockGraph:** `aws-access-key-id` / `aws-secret-access-key` under `/resonate/cdp-118203/blockgraph/`. Confirm exact param names before dev e2e. They are env-overridable via `BG_ACCESS_KEY_PARAM` / `BG_SECRET_KEY_PARAM` in the Lambda environment.
- **Thread pool concurrency:** All publish Lambdas process audiences concurrently. A single upload failure propagates — the state file is not written if any upload fails.
- **Formatter path layout:** The S3 partition order is `vendor=.../method=.../` (NOT the old `method=av/vendor=.../`). Using the old order causes "No files were able to be copied" errors (see CDP-118937).
- **SonarCloud per-Lambda projects:** Each Lambda has its own `sonar-project.properties`. SonarCloud coverage gates are enforced per PR. Target: >90% coverage on new code.
- **BlockGraph T05 dependency:** The rename and upload Lambdas (T07/T08) depend on T05 (Spark multi-column gzipped output) being available in the dev environment before full integration testing.
