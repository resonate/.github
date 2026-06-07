# resonate-terraform

## Project Purpose and Architecture Overview

This repository contains all Terraform/Terragrunt infrastructure-as-code for Resonate's AWS infrastructure, as well as Terraform for GitHub org management, Lacework security, and supporting operational scripts.

**All infrastructure changes are deployed through GitHub Actions — never run `terragrunt apply` locally.**

**Top-level layout:**

```
aws/
  global/           # Cross-account AWS resources (SQS, Batch, etc.)
    dev/            # Non-prod environment configs
    prod_us_east_2/ # Prod environment configs
modules/
  resources/        # Reusable Terraform modules
    lambda/         # Lambda function module (used by all Lambda repos)
    step_function/  # Step Function module (used by all pipeline repos)
    ...
github/             # Terraform for GitHub org (teams, repos, branch protection)
lacework/           # Terraform for Lacework security platform
scripts/            # Operational helper scripts (drift detection, etc.)
templates/          # Base templates updated by scripts
tests/              # Tests for Terraform templates
```

**Key module sources (referenced by other repos):**
- Lambda deployments: `git::ssh://github.com/resonate/resonate-terraform.git//modules/resources/lambda`
- Step Function deployments: `git::ssh://github.com/resonate/resonate-terraform.git//modules/resources/step_function`

These modules are pinned by tag in consuming repos' `terragrunt.hcl` files.

---

## Key Commands

### Drift Detection

```bash
# Run drift detection across all prod Terraform templates
python3 scripts/drift_detection.py

# Check known drift causes at:
# https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/3951329285/Terraform+Drift+Causes
```

### Terraform / Terragrunt (local plan only — applies must go through CI)

```bash
# Plan changes for a specific resource (non-prod)
cd aws/global/dev/<resource>
terragrunt plan

# Plan changes for prod (requires role assumption)
cd aws/global/prod_us_east_2/<resource>
terragrunt plan --terragrunt-iam-role arn:aws:iam::694585954309:role/ProdTerraform
```

---

## GitHub Actions Workflows

All deploys are triggered via GitHub Actions. The primary workflows are:

| Workflow | Trigger | What it deploys |
|---|---|---|
| `terraform-plan.yml` | PR to main | Runs `terragrunt plan` across changed directories |
| `terraform-apply.yml` | Push to main | Runs `terragrunt apply` for changed directories |
| `github-terraform.yml` | Push to main (github/ changes) | GitHub org Terraform |
| `lacework-terraform.yml` | Push to main (lacework/ changes) | Lacework Terraform |

**Infrastructure accounts:**
- Non-prod: default AWS account
- Prod: `arn:aws:iam::694585954309:role/ProdTerraform` (assumed via `role_arn` in root `terragrunt.hcl`)

**State buckets:**
- Non-prod: `resonate-terraforming-state`
- Prod: `resonate-terraforming-state-prod`

---

## Creating New Lambda Templates

Full process documented at:
https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/683540663/Lambda+Terraform+Creation+Deployment+Process

**Quick summary:**
1. Add a new directory under `modules/resources/` or use existing `lambda` module
2. Create a `terragrunt.hcl` in the consuming repo pointing to the module
3. Push to main — GitHub Actions applies the change

---

## Project-Specific Rules and Gotchas

- **Never apply locally.** All infra changes go through GitHub Actions. Local `terragrunt plan` is acceptable for verification, but never `apply`.
- **Terraform version:** Pinned to **1.5.7** in all workflows. Do not upgrade without coordinating with the team — version bumps require testing across all consuming repos.
- **Module versioning:** Other repos reference modules via `git::ssh://github.com/resonate/resonate-terraform.git//modules/resources/<module>?ref=<tag>`. When making breaking changes to a module, bump the version tag and update all consumers.
- **GitHub Terraform:** Changes to `github/` manage org-level settings (teams, repo permissions, branch protection). Test carefully — incorrect configs can lock people out of repos.
- **Drift:** Known drift exists in prod due to manual emergency changes. Always check the drift causes wiki before assuming a plan diff indicates a bug.

---

## Monitoring & Alerting Infrastructure (Recent — DE-13849, June 2026)

The monitoring stack was overhauled in June 2026, replacing Nagios-based alerting with a Slack + PagerDuty stack via AWS CloudWatch + SNS + Chatbot:

- **Slack channel:** `#monitoring-alerts-slack` receives CloudWatch alarms for EC2 status, Grafana ALB, synthetics canaries, and `pg_rtp` database
- **Chatbot scope:** Narrowed to only `monitoring-alerts-slack` channel (was previously broader)
- **PagerDuty:** Techops PagerDuty SNS topic referenced directly for Splunk dev EC2 status alarms
- **Decommissioned:** Nagios monitoring infrastructure removed; all active alarms now go through CloudWatch → SNS → Slack/PagerDuty

When adding new alarms, follow the pattern in `aws/global/dev/monitoring/` — publish to the relevant SNS topic (`monitoring-alerts-slack` for non-paging alerts, `techops-pagerduty` for paging alerts).
