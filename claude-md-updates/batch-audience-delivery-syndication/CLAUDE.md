# batch-audience-delivery-syndication

## Project Purpose and Architecture Overview

This repo contains Lambda functions and infrastructure for syndicated and custom **batch audience delivery** — the post-BEM/Stitch/Formatter step that uploads audience files to third-party platforms (OpenX, Viant, Experian, BlockGraph/FreeWheel, and others).

Each Lambda handles one stage of a vendor delivery workflow: config resolution → file renaming / format transformation → upload to vendor S3 / SFTP.

**Top-level layout:**

```
workflows/
  lambdas/                   # Python Lambda source — one subdir per function
    openx-publish-data-files/
    viant-publish-files/
    experian-syndication-notify/
    blockgraph-create-taxonomy-file/   # T06: generates BlockGraph metadata CSV
    blockgraph-rename-files/           # T07: concatenates + renames Spark output
    blockgraph-publish-files/          # T08: cross-account upload to BlockGraph S3
    ...
terraform/
  workflows/
    lambdas/                 # Terragrunt configs per Lambda per environment
      <lambda>/
        dev/
        prod/
        (qa/  where applicable)
.github/
  workflows/
    lambda.yml               # Manual deploy workflow for individual Lambdas
    all.yml                  # CI: auto-discovers all Lambda dirs and runs tests
```

---

## Key Commands

### Python Lambda Development

Each Lambda has its own `requirements.txt`. Use `pip` or `uv` to install dependencies per Lambda.

```bash
# Run tests for a specific Lambda (from the Lambda dir)
cd workflows/lambdas/<lambda-name>
pip install -r requirements.txt
python -m pytest tests/ -v

# Run all Lambda tests (from repo root)
for dir in workflows/lambdas/*/; do
  (cd "$dir" && python -m pytest tests/ -q 2>/dev/null || true)
done
```

### GitHub Actions Deployment

All deploys go through GitHub Actions. The `lambda.yml` workflow supports manual dispatch with an `environment` input.

```bash
# Deploy a specific Lambda (via gh CLI from your machine)
gh workflow run lambda.yml \
  -f lambda=blockgraph-publish-files \
  -f environment=dev \
  --ref <your-branch-name>

# Check deployment status
gh run list --workflow=lambda.yml --limit=5
gh run watch <run-id>
```

**Environments:** `dev`, `qa`, `prod` (not all Lambdas have all environments — check the `terraform/` dir).

---

## Lambda Inventory

### OpenX
- **`openx-publish-data-files`** — Uploads audience segment files to OpenX S3 (`resonate-openx-syndication` bucket). Outputs `*.csv.gz` files (hardcoded extension as of PR #46).

### Viant
- **`viant-publish-files`** — Publishes audience files to Viant's S3 via cross-account creds from SSM.

### Experian
- **`experian-syndication-notify`** — Notifies Experian after delivery.

### BlockGraph / FreeWheel (CDP-118694 epic)

The BlockGraph pipeline delivers RID-keyed (person-keyed) audience data to BlockGraph's S3 bucket using BG-issued cross-account credentials. Three Lambdas implement the delivery chain:

| Lambda | Ticket | Purpose |
|---|---|---|
| `blockgraph-create-taxonomy-file` | CDP-118915 (T06) | Generates metadata CSV(s): 13-field (initial/net-new) or 8-field (refresh/known) per BlockGraph spec. Reads audience set from ADS (syndicated) or event `audience_key_list` (custom). Routes by delivery state (known PSIDs). |
| `blockgraph-rename-files` | CDP-118916 (T07) | Concatenates per-audience Spark output parts into a single `resonate_<akey>_<ts>.csv.gz`. Uses S3 multipart copy for large files, download-concat-upload fallback for small parts. |
| `blockgraph-publish-files` | CDP-118917 (T08) | Uploads renamed segment files and metadata CSVs to BlockGraph's S3 (`auto/segment/upload/`, `auto/segment/metadata/`). Uses BG-issued cross-account credentials stored in SSM under a BlockGraph-specific prefix (see `terraform/workflows/lambdas/blockgraph-publish-files/` for authoritative parameter names); writes delivery-state delta (net-new PSIDs) to our own bucket. |

**BlockGraph delivery key facts:**
- Person-keyed (RID), not cookie-keyed (RCID) — audiences evaluated against a personJar bitmap
- Stitch table: `person_identity_graph_beta`; stitch columns: `norm_address_line, norm_city, norm_state, norm_zip, zip_plus4`
- Taxonomy metadata paths: `<prefix>/batch-delivery-payload/metadata/resonate_metadata_{initial,refresh}_<ts>.csv`
- State file path: `<prefix>/state/known-segments/run_date=YYYYMMDD/run_<ts>.csv`
- Two delivery modes: `blockgraph_syndicated` (ADS-sourced) and `blockgraph_custom` (event `audience_key_list`)
- SSM keys: BG-issued credentials stored under a BlockGraph-specific SSM prefix (see `terraform/workflows/lambdas/blockgraph-publish-files/` for authoritative names)

---

## Infrastructure Notes

- **IAM:** Each Lambda has its own execution role in `terraform/workflows/lambdas/<name>/<env>/`. The `blockgraph-publish-files` role has **no** direct permission on BlockGraph's S3 — writes happen via SSM-stored BG credentials.
- **No `terragrunt apply` locally** — all infra changes go through GitHub Actions.
- **Multipart upload abort:** `blockgraph-rename-files` calls `abort_multipart_upload` on failure so partial uploads don't accumulate.

---

## Formatter Path Layout

As of CDP-118857 / CDP-118937 (May 2026), the batch-expression-modeling Formatter outputs partitions in the order:
```
<prefix>/date=<date>/vendor=<vendor>/method=<method>/akey=<akey>/
```

Previous layout was `method=av/vendor=*/` — the swap caused "No files were able to be copied" errors in this repo's ASLs. Any ASL that constructs a `source_prefix` must use the new `vendor=*/method=*/` order.

---

## Testing

- Each Lambda's `tests/` directory uses `pytest` with `moto` or `unittest.mock` for S3/SSM simulation.
- `blockgraph-rename-files` uses an in-memory `FakeS3` that byte-checks gzip concatenation output.
- `blockgraph-create-taxonomy-file`: 18 unit tests (100% of ticket cases a–i).
- `blockgraph-rename-files`: 23 unit tests (100% line coverage).
- `blockgraph-publish-files`: 23 unit tests (94% coverage).
- CI (`all.yml`) auto-discovers all Lambda dirs and runs their test suites.
