## Recent Changes to Add to CLAUDE.md (batch-expression-modeling)

### 1. Add to "Common Lambda functions" in Python Lambda Development section:

- `batch-audience-delivery-processing-config` — now also supports **BlockGraph** as a delivery type (CDP-118913, Jun 2026). When BlockGraph is in the delivery schedule, the lambda generates BlockGraph-specific S3 paths and step configs.

### 2. Add new section under "Working with Step Functions" or as a new top-level section:

---

### AddJobFlowSteps Rate Limiting (batch-stitch)

The `batch-stitch-pipeline` Step Function's Map state uses `MaxConcurrency: 2` to rate-limit `AddJobFlowSteps` calls to EMR (CDP-118972). This prevents throttling when many stitch jobs are submitted in parallel. Do not increase this value without confirming EMR API rate limits.

---

### 3. Add to "Delivery Configuration" section — Important fields:

- `delivery_type` — now includes `"blockgraph"` in addition to `"daily"`, `"training"`, `"custom"`

---

### 4. Add new section: Batch Payload Formatters (CDP-118857, Apr–May 2026)

The `batch-delivery-formatter-pipeline` was extended with a **Batch Payload Formatters** migration that moved several vendors to use `CustomPayloadFormatter`:

- Vendors using `CustomPayloadFormatter` now receive `format=csv` in their formatter config.
- The `source_prefix` for custom-delivery configs was updated to match the new formatter path layout (CDP-118937). If you see missing files for a vendor, verify `source_prefix` matches the formatter's output path.
- Lambda test fixes were applied to handle the removal of Yahoo/Roku_direct/AEP vendors (PR #252).
- `formatter-metrics` Lambda was updated to emit metrics for the new formatter pipeline (PR #253).

---

### 5. Update GitHub Actions Workflows table:

| `vendor-delivery-metrics.yml` | push to `workflows/lambdas/vendor-delivery-metrics/**` | Deploys vendor-delivery-metrics Lambda with failure alerting |

(vendor-delivery-metrics Lambda was opted into Lambda failure alerts in PR #261, May 2026)
