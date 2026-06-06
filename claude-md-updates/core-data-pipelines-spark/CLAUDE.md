# core-data-pipelines-spark

Spark 3.5 / Scala 2.12 / JDK 17 upgrade of `core-data-pipeline`. Runs on EMR 7.12.

## Build

**JDK 17 is required for all builds and tests.**

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home
sbt compile       # compile
sbt test          # ~210+ tests across all suites
sbt assembly      # fat jar for EMR
```

## Key Paths

- Spark apps: `src/main/scala/com/resonate/spark/apps/`
- Shared utils: `src/main/scala/com/resonate/spark/utils/`
- UDAFs / UDFs: `src/main/scala/com/resonate/spark/functions/`
- Tests: `src/test/scala/com/resonate/spark/`

## Spark Apps

| App | Package | Notes |
|---|---|---|
| `SovrnLogMetrics` | `sampling` | Computes Sovrn overlap densities; migrated to Spark 3. Null-safe `meanDensities` helper (see below). |
| `CookieJarSampler` | `sampling` | **@deprecated** — `cookiejar-sample-export` pipeline decommissioned (Havas no longer a client). Code retained for reference only. |
| `TopicTagCookiejar` | `topictagmetrics` | Topic-tag sketch for cookiejar populations. |
| `TopicTagSketch` | `topictagmetrics` | Topic-tag sketch (ported from das-pipeline). |
| `TopicTagMetricsParcelsUpload` | `topictagmetrics` | Uploads tag parcels; uses 4-arg flat parcel layout (reverted from 6-arg in PR #28). |
| `BitmapSketch` | `bitmap` | Bitmap-backed sketch UDAFs. |

## Testing Patterns

- Tests use `QueryTest with SharedSparkSession` (Spark's built-in test framework)
- UDF tests: use companion objects to avoid serialization issues
- Spark 3 requires `spark.sql.legacy.allowUntypedScalaUDF=true` for old-style UDFs
- `from_unixtime` uses local timezone — use midday timestamps in tests
- `trim()` only strips spaces (0x20), not tabs
- Pre-existing flake in `CookieJarSamplerTest` (`should maintain correct counts after sampling`) — RNG tolerance issue, unrelated to any current changes, does not block merge

## Dependencies

- `das-expression-0.6.jar` — pure Java, JDK 8 bytecode, works on JDK 17 as-is. Provides `BitmapUtils` and `AudienceEvaluator`.
- `influxdb-java` — replaced `reactiveinflux_2.11`. Wrapper at `com.resonate.spark.utils.InfluxDBWriter`.
- Jackson pinned at 2.15.2 to match Spark 3.5.4 bundled version.

## Sovrn Spark 3 Notes (CDP-118269)

`SovrnLogMetrics` required two fixes for Spark 3 compatibility:

1. **Null-safe `meanDensities` helper** — In Spark 3, `DataFrame.summary("mean")` returns `null` for empty groups (Spark 2 returned `"NaN"`). The fix extracts the conversion into `meanDensities(mean: Row)` which maps `null → Double.NaN`.

2. **Empty-group density filter** — Drop zero-density groups before writing to InfluxDB to avoid downstream metric pollution.

## Rules

- **Tests first, then upgrade**: Any workflow must have unit + integration tests BEFORE deploying.
- **JDK 17 only**: Do not build or test with JDK 8.
- **Spot instances**: Every EMR workflow must use spot for core fleet with capacity-optimized allocation and on-demand fallback.
- **No hardcoded credentials**: `ParcelManager` had a dead branch with hardcoded IAM credentials — it was removed (security fix). Never add IAM keys to source code.
- **CookieJarSampler is deprecated**: Do not add new features or fix non-critical bugs in `CookieJarSampler`. It is kept for historical reference only.

## Related Repos

- `step-function-workflow-orchestrator` — EMR configs, step functions, deployment
- `core-data-pipeline` — original Spark 2.4 / Scala 2.11 repo (do not modify)
