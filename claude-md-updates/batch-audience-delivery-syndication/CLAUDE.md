# batch-audience-delivery-syndication

## Project Purpose and Architecture Overview

This repository manages the **batch audience delivery syndication** workflows — Lambda functions and Terraform infrastructure for distributing processed audience data to third-party partners. The initial use case is **BlockGraph/FreeWheel** file syndication.

The repo sits downstream of `batch-expression-modeling`: once BEM has evaluated audience expressions and produced output files, this repo's Lambda functions handle the final-mile delivery steps (rename files to partner conventions, generate taxonomy manifests, upload to partner SFTP/S3).

**Top-level layout:**

```
workflows/
  lambdas/
    blockgraph-rename-files/      # Renames BEM output part-files to BlockGraph naming convention
    blockgraph-create-taxonomy-file/  # Generates BlockGraph taxonomy CSV manifest
    blockgraph-publish-files/     # Uploads files to BlockGraph (FreeWheel SFTP or S3)
  step-functions/                 # Step Function ASL definitions (if any)
terraform/
  lambdas/
    blockgraph-rename-files/<env>/    terragrunt.hcl
    blockgraph-create-taxonomy-file/<env>/  terragrunt.hcl
    blockgraph-publish-files/<env>/   terragrunt.hcl
.github/
  workflows/                      # GitHub Actions for CI/CD
```

---

## Key Lambda Functions

### blockgraph-rename-files (CDP-118916)

Renames BEM output `part-N.csv.gz` files to the naming convention expected by BlockGraph/FreeWheel.

- Input: S3 prefix containing BEM output part files
- Output: Same S3 prefix with files renamed to partner convention
- Part-file regex is broad enough to match bare `part-N.csv.gz` patterns (not just `part-00000-*.csv.gz`)
- IAM: Least-privilege S3 permissions scoped to the delivery bucket

### blockgraph-create-taxonomy-file (CDP-118915)

Generates a BlockGraph taxonomy CSV manifest describing the audience segments delivered.

- Output S3 location for taxonomy file uses `SPI=N` (confirmed by product — not sensitive personal information)
- Does **not** use `ExpectedBucketOwner` verification (dropped after product confirmation)
- Runs SonarQube-clean code (all findings resolved)

### blockgraph-publish-files (CDP-118917)

Uploads renamed files and the taxonomy manifest to BlockGraph (FreeWheel's receiving endpoint).

- SSM Parameter Store is used for all secrets (SFTP credentials, API keys) — boto3 config follows standard SSM patterns
- Upload ordering: taxonomy file uploaded before data files (ensures manifest is present when partner processes data)
- IAM: Least-privilege — only the specific S3 paths and SSM parameters needed

---

## Development

### Running Lambda Tests

Each Lambda has its own test suite:

```bash
cd workflows/lambdas/<lambda-name>
pip install -r requirements.txt
python -m pytest tests/
```

### AWS Authentication

```bash
aws sso login
```

---

## Deployment

All deployments go through **GitHub Actions** — do not run `terragrunt apply` locally.

### Environments

| Environment | Purpose |
|---|---|
| `dev` / `dev2`-`devN` | Development testing |
| `qa` | QA/staging environment |
| `prod` | Production |

### Workflow Dispatch

```bash
# Deploy a specific Lambda to an environment
gh workflow run lambdas.yml -f lambda=blockgraph-publish-files -f environment=qa --ref <your-branch>
```

### Terraform Module Source

Lambda infrastructure uses the shared module from `resonate-terraform`:
```
git::ssh://github.com/resonate/resonate-terraform.git//modules/resources/lambda
```

---

## Project-Specific Rules and Gotchas

- **SPI classification:** BlockGraph delivery files are classified `SPI=N` — confirmed with product. Do not add `ExpectedBucketOwner` checks unless this changes.
- **Upload ordering matters:** Taxonomy file must be uploaded before data files. The `blockgraph-publish-files` Lambda enforces this order — do not parallelize these uploads.
- **Part-file naming:** The rename regex matches bare `part-N.csv.gz` (not just Spark's `part-00000-uuid-*.csv.gz`). Test with actual BEM output when changing the regex.
- **SSM secrets:** All credentials are in Parameter Store under a `/blockgraph/` prefix. Keys are environment-specific — dev uses dev credentials, prod uses prod credentials.
- **Downstream of BEM:** This repo processes output from `batch-expression-modeling`. If BEM changes its output format or S3 path structure, update the input configuration here accordingly.

---

## End-to-End BlockGraph Flow

```
batch-expression-modeling
  └─> BEM evaluates expressions → Stitch joins with person_identity_graph_beta
        └─> Output files in S3 (part-N.csv.gz)
              └─> batch-audience-delivery-syndication
                    ├─> blockgraph-rename-files  (rename to partner convention)
                    ├─> blockgraph-create-taxonomy-file  (generate manifest)
                    └─> blockgraph-publish-files  (upload to FreeWheel)
```

To locate the step function that orchestrates these three Lambdas, search for references to `blockgraph-rename-files`, `blockgraph-create-taxonomy-file`, or `blockgraph-publish-files` in the ASL JSON files under `step-function-workflow-orchestrator/pipelines/` or under `workflows/step-functions/` in this repo.
