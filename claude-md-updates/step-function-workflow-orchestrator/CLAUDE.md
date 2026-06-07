# step-function-workflow-orchestrator

## Project Purpose and Architecture Overview

This repo contains the X-Men team's AWS Step Function pipelines and the supporting Lambda functions that power the Connected Profiles data platform. It is **not** a single application — it is a collection of ~50 independent pipelines (state machines) and ~30 Lambda functions grouped by runtime.

**Top-level layout:**

```
pipelines/          # Step Function definitions — one subdir per pipeline
  <pipeline>/
    statemachine/   # ASL JSON state machine definition
    config/         # EMR cluster configs + per-environment params.json
    events/         # EventBridge event payloads per environment
    tests/          # Unit + integration tests
terraform/
  pipelines/        # Terragrunt config for deploying step functions
    <pipeline>/<env>/terragrunt.hcl
  python_lambdas/   # Terragrunt config for Python lambdas
  java_lambdas/     # Terragrunt config for Java lambdas
  javascript-lambdas/
  lambda.hcl        # Shared lambda Terragrunt root config
  step_function.hcl # Shared step function Terragrunt root config
python_lambdas/     # Python Lambda source (one subdir per function)
java_lambdas/       # Java Lambda source (dos-lambdas, metrics-formatter)
javascript-lambdas/ # Node.js Lambda source (human-approval-*)
scripts/            # Operational helper scripts (Python, not deployed)
```

**Infrastructure accounts:**
- Non-prod: default AWS account (no `--profile` flag needed)
- Prod: `arn:aws:iam::694585954309:role/ProdTerraform` (assumed via `role_arn` in `step_function.hcl`)

**State bucket:** `resonate-terraforming-state` (non-prod) / `resonate-terraforming-state-prod` (prod)
**State key prefixes:**
- Lambdas: `environments/mgmt/lambdas/x_men/`
- Step Functions: `environments/mgmt/lambdas/x_men|agents_of_shield/` (the `|` is a literal character in the S3 key, not an "or")

---

## Key Commands

### Python Lambdas

All Python lambdas live under `python_lambdas/<name>/`. Dependencies are per-lambda `requirements.txt` files. The workspace-level dev tooling uses `uv` (see `python_lambdas/pyproject.toml`).

```bash
# Install dev deps (from python_lambdas/)
uv sync

# Run unit tests for a specific lambda
cd python_lambdas/<lambda-name>
pytest tests/

# Run unit tests across all Python lambdas (from python_lambdas/)
pytest

# Lint
flake8 python_lambdas/

# Build/run a single lambda locally with SAM
cd python_lambdas/<lambda-name>
sam build --use-container
sam local invoke --event events/event.json
```

### Java Lambdas

```bash
# Build dos-lambdas fat JAR (shadow JAR)
cd java_lambdas/dos-lambdas
./gradlew shadowJar

# Build metrics-formatter fat JAR
cd java_lambdas/metrics-formatter
./gradlew shadowJar

# Run tests
./gradlew test
```

### JavaScript Lambdas

```bash
# No build step — plain Node.js
cd javascript-lambdas/human-approval-emailer
npm install

cd javascript-lambdas/human-approval-callback
npm install
```

### Integration Tests (Step Functions)

Integration tests start the real Step Function in AWS and poll to completion. They require AWS credentials for the dev/integration environment.

```bash
cd pipelines/<pipeline>/tests/integration
pytest --state-machine-name <pipeline>-integration
# e.g.
pytest --state-machine-name behavior-stitch-integration
```

### Operational Scripts

```bash
# Generate environment-specific configs from integration config (run from scripts/)
python3 generate_environment_configs.py <env> [path-to-pipelines]
# e.g.
python3 generate_environment_configs.py dev6

# Mark DOS unprocessed files as processed (recovery helper, run from scripts/)
python3 mark_dos_unprocessed_as_processed.py \
  --onboard_path s3://... \
  --completion_status_path s3://...
```

---

## Terraform / Infrastructure Specifics

All deployments go through **GitHub Actions** — there are no local `terragrunt apply` commands in normal workflow. Use the Actions UI (or `workflow_dispatch`) for all deploys.

### GitHub Actions Workflows

| Workflow | Trigger | What it deploys |
|---|---|---|
| `step_function.yml` | push to `pipelines/**` or manual | Step Functions via `terraform/pipelines/<sf>/<env>/` |
| `python_lambda.yml` | push to `python_lambdas/**` or manual | Python Lambdas via `terraform/python_lambdas/<lambda>/<env>/` |
| `java_lambda.yml` | push to `java_lambdas/**` or manual | Java Lambdas via `terraform/java_lambdas/<lambda>/<env>/` |
| `nodejs_lambda.yml` | push to `javascript-lambdas/**` or manual | JS Lambdas via `terraform/javascript-lambdas/<lambda>/<env>/` |
| `unzip-copy.yml` | push to `common/unzip-copy/**` or manual | Docker image → ECR → ECS task definition |

**Supported environments by workflow:**
- `step_function.yml`: `dev`, `dev2`–`dev9`, `integration`, `qa`, `prod`
- `python_lambda.yml`: `dev`, `dev2`–`dev9`, `nonprod`, `qa`, `prod`
- `java_lambda.yml` / `nodejs_lambda.yml`: varies per pipeline's deployed env folders
- `unzip-copy.yml`: `dev`, `prod` only

### Terragrunt Module Sources

- **Step functions:** `git::ssh://github.com/resonate/resonate-terraform.git//modules/resources/step_function`
- **Lambdas:** `git::ssh://github.com/resonate/resonate-terraform.git//modules/resources/lambda`
- Terraform version pinned to **1.5.7** in all workflows.

### Pipeline Config Layout

Each pipeline has per-environment configs at `pipelines/<name>/config/<env>/params.json` and shared configs at `pipelines/<name>/config/emr.json`, `emr-on-demand.json`, `steps.json`. The `generate_environment_configs.py` script derives `dev6` (or other dev variants) configs from the `integration` config by rewriting S3 bucket suffixes.

### Lambda ARN Injection

State machine ASL files use template variables like `${DynamicDatesFunctionArn}`. These are resolved at deploy time via `lambda_replacement_vars` in each pipeline's `terragrunt.hcl`.

### unzip-copy is Different

`python_lambdas/unzip-copy` is deployed as a **Docker container to ECS** (not as a Lambda ZIP). It has its own `Dockerfile`, `taskdef-dev.json` / `taskdef-prod.json`, and a dedicated `unzip-copy.yml` workflow.

---

## Project-Specific Rules and Gotchas

- **Never run `terragrunt apply` locally** — all infra changes must go through GitHub Actions CI/CD.
- **Config drift in dev variants:** When creating a new dev environment (e.g. `dev7`), run `scripts/generate_environment_configs.py dev7` to generate correct S3 bucket paths — do not hand-edit `params.json` copies.
- **S3 bucket naming convention:** Non-prod buckets append `-<env>` (e.g. `resonate-core-datasets-dev`). The `generate_environment_configs.py` script handles rewriting these automatically.
- **Lambda ARN template variables** (`${XxxFunctionArn}`) in ASL JSON are **not** real ARNs — they are substituted by Terraform at deploy time using `lambda_replacement_vars` in terragrunt.hcl.
- **Prod-only step function scheduling:** Step functions are only enabled (`is_enabled = true`) in prod environments; all other environments deploy the state machine but leave the EventBridge schedule disabled.
- **Python runtime:** Local dev tooling requires Python 3.13+ (enforced by `python_lambdas/pyproject.toml`). The AWS runtime deployed per-lambda varies: currently `python3.7`, `python3.9`, `python3.11`, `python3.12`, or `python3.13` depending on the function — check the relevant `terraform/python_lambdas/<name>/terragrunt.hcl`. Dev tooling uses `uv`; individual lambdas declare their own `requirements.txt`.
- **Java lambdas use shadow JARs:** The Gradle `shadowJar` task produces the fat JAR uploaded to Lambda. Do not use the plain `jar` task.
- **Integration tests are long-running:** The behavior-stitch integration test has a 2-hour timeout (`EXECUTION_TIMEOUT = 7200`). Do not run locally unless you intend to wait.
- **SNS topics for failure/pass alerts:** Pipelines use `sns_topic_replacement_vars` in each environment's `terragrunt.hcl` to inject `TopicFailArn`, `TopicPassArn`, and (where applicable) `TopicWarnArn`. Topic names are environment-specific: dev environments typically use `datapipeline-fail-qa-topic` / `datapipeline-pass-qa-topic`, dev6 uses `datapipeline-fail-dev6-topic` / `datapipeline-pass-dev6-topic`, and prod pipelines use pipeline-specific topics (e.g. `dos-datapipeline-fail-prod-topic`, `modeling-team-pagerduty`). These SNS topics are managed outside this repo.

---

## Recent Changes (as of June 2026)

### EMR7 Upgrades (CDP-118269)
Multiple pipelines were upgraded from EMR6 to EMR7. If you're working on a pipeline and its `emr.json` / `emr-on-demand.json` references an old release label, check the corresponding PR for the correct EMR7 release label. Affected pipelines include:
- `maid-onboarder`
- `topic-aggregation`
- `idsync-overlap`
- `marketops-overlap`
- `behavior-stitch`

Each upgrade also updated the jar S3 bucket path for dev environments (`integration` → `dev` bucket suffix).

### Decommissioned: cookiejar-sample-export (June 2026)
The `cookiejar-sample-export` Lambda **infrastructure has been removed** (Terraform config deleted), but the source code under `python_lambdas/cookiejar-sample-export/` is intentionally kept for reference (marked deprecated). Do not redeploy this Lambda — the business use case has been retired. If you need to clean up, only the code directory remains.
