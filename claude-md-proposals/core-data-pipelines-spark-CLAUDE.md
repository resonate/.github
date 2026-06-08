# core-data-pipelines-spark

Spark 3.5 / Scala 2.12 / JDK 17 upgrade of `core-data-pipeline`. Runs on EMR 7.12.

## Build

**JDK 17 is required for all builds and tests.**

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home
sbt compile       # compile
sbt test          # 282 tests, 66 suites
sbt assembly      # fat jar for EMR
```

## Key Paths

- Spark apps: `src/main/scala/com/resonate/spark/apps/`
- Shared utils: `src/main/scala/com/resonate/spark/utils/`
- UDAFs: `src/main/scala/com/resonate/spark/functions/`
- Tests: `src/test/scala/com/resonate/spark/`

## Testing Patterns

- Tests use `QueryTest with SharedSparkSession` (Spark's built-in test framework)
- UDF tests: use companion objects to avoid serialization issues
- Spark 3 requires `spark.sql.legacy.allowUntypedScalaUDF=true` for old-style UDFs
- `from_unixtime` uses local timezone — use midday timestamps in tests
- `trim()` only strips spaces (0x20), not tabs

## Dependencies

- `das-expression-0.6.jar` — pure Java, JDK 8 bytecode, works on JDK 17 as-is. Provides `BitmapUtils` and `AudienceEvaluator`.
- `influxdb-java` — replaced `reactiveinflux_2.11`. Wrapper at `com.resonate.spark.utils.InfluxDBWriter`.
- Jackson pinned at 2.15.2 to match Spark 3.5.4 bundled version.

## Rules

- **Tests first, then upgrade**: Any workflow must have unit + integration tests BEFORE deploying.
- **JDK 17 only**: Do not build or test with JDK 8.
- **Spot instances**: Every EMR workflow must use spot for core fleet with capacity-optimized allocation and on-demand fallback.

## Deprecated Apps

- **`CookieJarSampler`** (`com.resonate.spark.apps.sampling.CookieJarSampler`) — marked `@deprecated` as of 2026-06. The `cookiejar-sample-export` pipeline it powered has been decommissioned (Havas is no longer a client; infra removed in `step-function-workflow-orchestrator` #748). The Scala code is kept for historical reference only. Do not add new references or submit this class to EMR.

## Spark 3 Migration Notes (Sovrn)

- **`summary()` null-safe mean conversion** — Sovrn pipeline uses `summary()` to compute statistics. Spark 3 changed the return type of `summary()` mean column from `Double` to `String`. Use null-safe conversion (`toDoubleOption` or explicit cast with null check) when extracting mean values from `summary()` output. Plain `.toDouble` throws `NumberFormatException` on null summary rows (CDP-118269).

## Related Repos

- `step-function-workflow-orchestrator` — EMR configs, step functions, deployment
- `core-data-pipeline` — original Spark 2.4 / Scala 2.11 repo (do not modify)
