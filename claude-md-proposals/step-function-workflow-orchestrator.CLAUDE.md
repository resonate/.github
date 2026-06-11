# CLAUDE.md update — resonate/step-function-workflow-orchestrator
# Append the following three sections to the END of the existing CLAUDE.md

---

## Decommissioned Pipelines (Removed from Deployment)

These pipelines have had their Terraform infrastructure removed and **must not be redeployed**.
Pipeline code and config files are retained for historical reference only.

| Pipeline | Reason | PR |
|---|---|---|
| `fusion-behavior-preprocess` | Dormant since 2023; inadvertently re-deployed 2026-05-28. Sovrn/Havas no longer a client. | #734 |
| `cookiejar-sample-export` | Havas no longer a client; `CookieJarSampler` Scala app marked `@deprecated` in `core-data-pipelines-spark`. | #748 |

Both pipelines still have AWS resources (Step Functions, EventBridge rules, IAM roles) requiring manual `terragrunt destroy` or console cleanup in both dev and prod accounts. See `DEPRECATED.md` inside each pipeline directory.

---

## EMR 7.12.0 / Spark 3 Migration (CDP-118269)

These pipelines have been migrated from `emr-5.34.0` + Spark 2 to `emr-7.12.0` + Spark 3:

| Pipeline | PR | Notes |
|---|---|---|
| `sovrn-weekly` | #729 | |
| `maid-onboarder` | #730 | Prod-volume validated end-to-end |
| `damlam-preparation` | #732 | |
| `idsync-overlap` | #733 | YARN vcore fix: `yarn.nodemanager.resource.cpu-vcores = 16` on integration |
| `behavior-stitch` | #737 | YARN vcore fix applied; use `--master yarn` for Spark 3 |
| `marketops-overlap` | #731 | |
| `topic-aggregation` | #736 | Requires `--master yarn` flag in Spark 3 spark-submit args |

**Jar convention:** Spark 3 pipelines use `core-data-pipeline-spark3-latest.jar`; legacy pipelines use `core-data-pipeline-latest.jar` or `assembly-*.jar`. When migrating a new pipeline update both `params.json` (prod + integration) and `emr.json` (prod + integration + on-demand variants).

**YARN memory on EMR 7 (r5.xlarge):** EMR 7 overhead factor ~18.8% (higher than EMR 5). Reduce integration executor memory (e.g. `21g → 16g`). Prod r5d.4xlarge can keep 21G.

**Dev jar path:** Dev environments must reference the `-dev` S3 bucket (e.g. `resonate-core-datasets-dev/...`), not the prod bucket.

---

## Geo-Location Pipeline Recent Changes

### TapAd-Derived HEM→RCID Stitch (CDP-118512)
`l2-processing` pipeline now ingests TapAd Digital Graph + LiveRamp Tradedesk Agg to supplement LiveIntent-based HEM→RCID derivation (~+21% coverage lift on district-attributed RCIDs). The `dynamic-dates` Lambda resolves TapAd's flat-file dates (no date-named subdirectory) via a new flat-file sentinel mode.

### Zip→District Mappings (CDP-118946)
Four new CSVs under `pipelines/geo-location/config/zip-district-mappings/` (one per district type: congress, proposed-congress, state-senate, state-house). Schema: `zip, state, <district-type>`. Deployed to S3 via terragrunt `configurations` entries. Passed as `zipDistrictMappingsBasePath` to both `GeoLocationDaily` and `GeoLocationFullBackfill`.

### Backfill `Should Run Full` Bug Fix (CDP-118512)
A bug in `geo_location.asl.json` where a pre-existing geo full for today overrode the backfill output was fixed by adding an `IsPresent` check on `$.FullInputPath` before the path-equality choice. If backfill output is not propagating, check the `Should Run Full` state machine choice state.
