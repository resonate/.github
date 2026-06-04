# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**identity-graph** is a Scala 2.12 / Spark 3.5.0 project that builds and maintains Resonate's identity graph — linking RCIDs to vendor identity keys across multiple data sources (TransUnion, Sovrn, Experian, data append, etc.). It also contains the **PRISM** (Person-level Resolution & Identity Stitching Model) project for person-level identity resolution.

## Repository Structure

```
src/main/scala/
  com/resonate/identitygraph/
    IdentityGraphJob.scala        # Main identity graph job
    PersonIdentityJob.scala       # PRISM person-level identity job (added May 2026)
    ExperianPreprocessJob.scala   # Experian data preprocessing (added Apr/May 2026)
    metrics/                      # InfluxDB metric emission
docs/
  prism/                          # PRISM design docs, project plan, reference artifacts (added Jun 2026)
build.sbt                         # Scala 2.12 / Spark 3.5.0
```

## Common Development Tasks

### Build and Test

```bash
# Compile
sbt compile

# Run tests
sbt test
# Note: tests require JDK 17+ — build.sbt adds --add-opens flags automatically for JDK 9+.
# Use: fork in Test := true (already set in build.sbt)

# Package JAR
sbt assembly
```

### AWS Authentication

```bash
aws sso login
```

Required before any S3 operations.

### Deployment

Production JAR path: `s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar`

Deployments go through GitHub Actions. See the [GitOps docs](https://resonate-jira.atlassian.net/wiki/spaces/ASD/pages/3683057840/Pipeline+Scala+SBT+GitOps).

## Key Concepts

### Identity Graph Job

`IdentityGraphJob` reads data from multiple identity sources and produces a unified stitch table mapping RCIDs to vendor identity keys. Data sources include:
- TransUnion aggregate data (TU monthly/aggregate — columns renamed in Jul 2024)
- Sovrn IP and geo enrichment
- Data append (reads from Snowflake via `spark-snowflake` connector, added Sep 2025)
- HEMs from TransUnion (added Oct 2024)

**MAID ordering is deterministic** (CDP-113865): tie-breaking logic ensures consistent output across runs.

### PRISM — Person-level Identity Resolution (added May–Jun 2026)

PRISM is a new initiative for person-level identity resolution. The `PersonIdentityJob` was added in May 2026 (PR #22) and PRISM design documents, project plan, and reference artifacts were committed in Jun 2026 (PR #25).

Key details:
- Design docs live in `docs/prism/`
- Address-only handling is documented (decision cross-linked with the Append decision pages)
- `PersonIdentityJob` includes null guards and deterministic tie-breakers

### Experian Data Processor (added Apr–May 2026, CDP-118890)

`ExperianPreprocessJob` preprocesses Experian data for use in the identity graph. This was added in PR #21 (SayaliPat). It was also integrated into `batch-expression-modeling` for custom audience delivery (CDP-118793/CDP-118833).

### Snowflake Connector

Data append data is read from Snowflake using `net.snowflake:spark-snowflake:2.16.0-spark_3.3`. The Snowflake connection is configured via standard Snowflake Spark options. Reading was changed from an S3 path to Snowflake in Sep 2025 (PR #16).

### JDK Compatibility

`build.sbt` conditionally adds `--add-opens` JVM flags for JDK 9+. Tests must run with `fork := true` (already configured). The CI uses JDK 17. Tests run in isolation (`parallelExecution in Test := false`).

## Project-Specific Rules and Gotchas

- **Deterministic MAID ordering**: Always maintain deterministic tie-breakers in any join/aggregation that produces MAID output — flapping output breaks downstream consumers.
- **TransUnion column names**: Columns use `tu_aggregate` naming (not `tu_monthly`) since Jul 2024 — do not revert.
- **Data append reads from Snowflake**, not S3 — the Snowflake connector (`spark-snowflake`) must be on the classpath.
- **PersonIdentityJob null guards**: The PRISM person identity logic has explicit null guards and deterministic tie-breakers added in review — do not remove them.
- **Sovrn path**: Reading Sovrn data uses a specific updated path (changed in PR #14, Oct 2024) — verify the path before modifying Sovrn-related code.
- **PAT-based token** is used for GitHub Actions auth (updated PR #18) — do not revert to app-based auth.
