# CLAUDE.md Updates

This directory contains proposed CLAUDE.md files for 5 repositories that had active PRs merged into main (May–June 2026) by team members: SayaliPat, shrivastavakapil2000, JoeVsVolcano, mike-brant, nathan-resonate.

## Files to Apply

Each subdirectory contains a `CLAUDE.md` to be committed to the root of the corresponding repository:

| Directory | Target Repository | Action |
|-----------|------------------|--------|
| `step-function-workflow-orchestrator/` | `resonate/step-function-workflow-orchestrator` | **Create** new CLAUDE.md |
| `batch-expression-modeling/` | `resonate/batch-expression-modeling` | **Replace** existing CLAUDE.md |
| `identity-graph/` | `resonate/identity-graph` | **Create** new CLAUDE.md |
| `batch-audience-delivery-syndication/` | `resonate/batch-audience-delivery-syndication` | **Create** new CLAUDE.md |
| `dos-data-pipeline/` | `resonate/dos-data-pipeline` | **Create** new CLAUDE.md |

## How to Apply

For each repository, create a branch and open a PR:

```bash
# 1. Check out the target repo
cd /path/to/step-function-workflow-orchestrator
git checkout -b chore/add-claude-md

# 2. Copy the file from this repo (resonate/.github)
#    Assumes resonate/.github is cloned alongside the target repo
cp ../resonate-.github/claude-md-updates/step-function-workflow-orchestrator/CLAUDE.md ./CLAUDE.md
# Or download directly from GitHub:
# curl -o CLAUDE.md https://raw.githubusercontent.com/resonate/.github/main/claude-md-updates/step-function-workflow-orchestrator/CLAUDE.md

git add CLAUDE.md
git commit -m "chore: add CLAUDE.md with project guidance for Claude Code"
git push -u origin chore/add-claude-md
# Then open PR via GitHub UI or: gh pr create --title "chore: add CLAUDE.md"
```

## What's Covered in Each File

### step-function-workflow-orchestrator
- Pipeline inventory (12 active pipelines)
- EMR 5→7 migration notes (Spark 2→3, yarn vcore fix)
- Integration test patterns
- Dynamic dates lambda (directory mode vs flat-file sentinel mode)
- Recent changes: EMR migrations, QA envs, fusion-behavior-preprocess removal, geo district namespace fixes

### batch-expression-modeling (UPDATE)
- All existing content preserved
- Added: formatter metrics lambda, stitch throttle protection (MaxConcurrency=2)
- Added: `delta_with_full_fallback` refresh type handling
- Added: Formatter output path layout (post CDP-118857 partition order change)

### identity-graph
- 11 Spark pipeline jobs and their purposes
- Shared utilities (HashUtils, StagingWriter, AddressNormalizer, IpFilter, ScoringConfig)
- PRISM design overview and 6 tracks
- All jobs use scopt CLI args
- Recent changes: port from resonate-research, ExperianDataProcessor, PRISM docs

### batch-audience-delivery-syndication
- Supported vendors (OpenX, Experian, Viant, BlockGraph)
- openx-publish-data-files: hardcoded .csv.gz extension (DO NOT revert to dynamic parsing)
- blockgraph-create-taxonomy-file: taxonomy generation, SPI=N constant
- Source path partition order (vendor=*/method=av/)

### dos-data-pipeline
- district_source provenance (L2_CONFIRMED, L2_UNCONFIRMED, IP_INFERRED)
- IP-inferred district fallback via 4 ZIP→district CSVs
- ToBitmap gating on L2_CONFIRMED
- ZIP→district namespace requirements (L2 canonical vs floterial)
- GeoLocationFullBackfill: always re-derives all 4 districts
