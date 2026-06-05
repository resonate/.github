# CLAUDE.md update — resonate/batch-expression-modeling
# Append the following section to the END of the existing CLAUDE.md

---

## Vendor Configuration (`vendor-config.ini`) — Recent Additions

File: `workflows/lambdas/batch-audience-delivery-config/vendor-config.ini`

### New Config Keys

| Key | Default | Purpose |
|---|---|---|
| `stitch_columns` | — | Comma-separated multi-column stitch (use instead of `stitch_column` for vendors that join on multiple fields, e.g. BlockGraph address). |
| `audience_bitmap_path` | `tagav` | Bitmap source for audience evaluation. Set to `person_jar` for person-keyed (RID) vendors like BlockGraph. |

All existing single-column vendors continue to use `stitch_column` (singular). `get_audience_bitmap_path(delivery_type)` returns the configured source (default `tagav`).

### BlockGraph Delivery (CDP-118913)

BlockGraph is **person-keyed (RID)**, not cookie-keyed (RCID). Two new sections in `vendor-config.ini`:

- `[blockgraph_syndicated]` — syndication delivery via normalized RID→ADDRESS table
- `[blockgraph_custom]` — custom delivery, same stitch shape

Key differences from cookie-keyed vendors:
- `stitch_columns = norm_address_line,norm_city,norm_state,norm_zip,zip_plus4` — normalized address stitch (5 columns)
- `stitch_table_name = person_identity_graph_beta` — beta RID→ADDRESS table
- `audience_bitmap_path = person_jar` — evaluates audiences against the personJar bitmap (not tagav)
- `person_jar_path` is supplied per-run in the delivery event (`delivery_config.person_jar_path`)

### Formatter Output Path Layout Change (CDP-118857 / CDP-118937)

The batch-delivery-formatter flipped partition order from `date=*/method=av/vendor=*/akey=*` to `date=*/vendor=*/method=*/akey=*`. All downstream consumer state machines were updated accordingly:
- `batch-audience-custom-delivery` (#256)
- `batch-audience-delivery-syndication` (yahoo, experian, openx/viant variants)

If you see `No files were able to be copied` errors after a formatter run, check that the consumer ASL's `source_prefix` uses `vendor=*/method=*` order.

### Formatter Metrics Lambda (CDP-118857)

`batch-delivery-formatter-pipeline` now invokes `batch-delivery-formatter-publish-metrics` Lambda after each successful EMR run. Lambda ARN injected via `PublishMetricsFunctionArn` in each pipeline's `terragrunt.hcl`.

Metrics published to InfluxDB:
- `com.resonate.delivery.format.count` — per-audience row counts
- `com.resonate.delivery.format.aggregate` — aggregate metrics (vendor + method + delivery_type tags, no audience_key)

### Batch Stitch Throttling (CDP-118972)

`batch-stitch-pipeline` uses `MaxConcurrency=2` on the stitch Map state to prevent bursting `AddJobFlowSteps` API calls. A stagger wait of `(Map.Item.Index × wait_seconds)` is inserted before each step submission.
