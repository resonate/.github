# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **Batch Expression Modeling (BEM)** system for Resonate's audience delivery platform. It evaluates audience expressions against user behavioral data at scale, processes the results through identity stitching, and formats the output for various downstream vendors (Meta, TikTok, TheTradeDesk, etc.).

### High-Level Architecture

The system consists of three main components orchestrated by AWS Step Functions:

1. **BEM (Batch Expression Modeling)** - Scala/Spark job that evaluates ~350 audience expressions against 2TB+ of SuperTagAv data
2. **Stitch** - Scala/Spark job that joins evaluated expressions with vendor identity tables (cookies, HEMs, device IDs)
3. **Formatter** - Scala/Spark job that formats stitched data for specific vendor requirements

Each component runs on AWS EMR clusters and is configured/orchestrated by Lambda functions and Step Functions state machines.

## Repository Structure

```
├── src/main/scala/com/resonate/
│   ├── bem/                    # Batch Expression Modeling (evaluates expressions against SuperTagAv)
│   ├── stitch/                 # Identity stitching (joins RCID with vendor IDs)
│   ├── delivery/               # Delivery formatting and vendor-specific logic
│   └── utils/                  # Shared utilities
├── workflows/
│   ├── lambdas/                # Lambda functions for pipeline configuration
│   │   ├── batch-audience-delivery-processing-config/  # Main config lambda
│   │   ├── batch-vendor-stitch/                        # Stitch config lambda
│   │   ├── bem-gen-job-parameters/                     # BEM job parameters
│   │   └── ...
│   └── step-functions/         # Step function state machines (ASL JSON)
│       ├── batch-audience-delivery-processing/         # Main pipeline
│       ├── batch-audience-delivery-daily-pipeline/     # Daily orchestrator
│       ├── bem-pipeline/                               # BEM execution
│       ├── batch-stitch-pipeline/                      # Stitch execution
│       └── batch-delivery-formatter-pipeline/          # Formatter execution
├── terraform/                  # Infrastructure as code (one directory per Lambda/Step Function)
├── integration-tests/          # End-to-end integration tests
└── build.sbt                   # SBT build configuration
```

## Common Development Tasks

### Scala/Spark Development

**Build and test:**
```bash
# Compile Scala code
./compile.sh  # or: sbt compile

# Run tests
sbt test

# Run specific test
sbt "testOnly com.resonate.bem.BatchExpressionModelJobTest"

# Package JAR for EMR deployment
./package.sh  # or: sbt assembly
# Output: target/batch-expression-modeling.jar
```

**AWS authentication:**
```bash
# Required before uploading JARs to S3 or accessing AWS resources
aws sso login
```

### Python Lambda Development

**Lambda functions are in `workflows/lambdas/`**. Each has its own `requirements.txt`.

**Run tests for a Lambda:**
```bash
cd workflows/lambdas/<lambda-name>
python -m pytest tests/
```

**Common Lambda functions:**
- `batch-audience-delivery-processing-config` - Generates pipeline configuration from delivery schedule
- `bem-gen-job-parameters` - Generates EMR step parameters for BEM job
- `batch-vendor-stitch` - Generates stitch configuration per vendor

### Integration Tests

**Run full integration test:**
```bash
cd integration-tests

# First time: install dependencies
pip install -r requirements.txt

# Run setup + execution + validation
bash run_full_integration_test.sh
```

The integration test:
1. Sets up test data in S3 with datetime-based paths (e.g., `integration/data/20250914_143022/`)
2. Triggers the `aud-delivery-proc-integration` step function
3. Polls for completion and validates SUCCESS status

**Cost:** ~$0.25-0.30 per run, ~15-25 minutes with minimal test data.

### GitHub Actions Workflows

**Workflow files (in `.github/workflows/`):**

| Workflow File | Description |
|--------------|-------------|
| `bem-jar.yml` | Builds and publishes the Scala JAR to S3 |
| `bem-lambdas.yml` | Deploys Lambda functions |
| `bem-step-functions.yml` | Deploys Step Functions |
| `bem-unit-tests.yml` | Runs Scala unit tests |

**Deploy to an environment:**

⚠️ **CRITICAL: Always specify `--ref` when deploying from a feature branch!**

Without `--ref`, `gh workflow run` deploys from the **default branch (main)**, NOT your current branch. This is a common mistake that results in deploying old code.

```bash
# Deploy JAR (required input: environment)
# JAR goes to qa (integration environment uses qa JAR)
gh workflow run bem-jar.yml -f environment=qa --ref <your-branch-name>

# Deploy Lambdas (required input: environment)
gh workflow run bem-lambdas.yml -f environment=integration --ref <your-branch-name>

# Deploy Step Functions (required inputs: environment, deployment_type)
gh workflow run bem-step-functions.yml -f environment=integration -f deployment_type=All --ref <your-branch-name>
```

**Example deploying all components from a feature branch:**
```bash
BRANCH="feature/CDP-123456-my-feature"
gh workflow run bem-jar.yml -f environment=qa --ref $BRANCH
gh workflow run bem-lambdas.yml -f environment=integration --ref $BRANCH
gh workflow run bem-step-functions.yml -f environment=integration -f deployment_type=All --ref $BRANCH
```

**Step Functions deployment_type options:**
- `All` - Deploy all step functions
- `BEM Pipeline Step Function`
- `Batch Stitch Pipeline`
- `Batch Delivery Formatter Pipeline`
- `Audience Delivery Pipeline`
- `Audience Delivery Custom Delivery Pipeline`
- `Audience Delivery Processing Pipeline`
- `Audience Delivery Daily Pipeline`
- `Batch Delivery Formatter Metrics Lambda` *(added CDP-118857)*

**Environment values:**
- `integration` - Integration testing environment (uses QA JAR)
- `dev`, `dev2`-`dev7` - Development environments
- `nonprod` - QA/staging environment
- `prod` - Production environment

**Check deployment status:**
```bash
# List recent runs for a workflow
gh run list --workflow=bem-jar.yml --limit=3
gh run list --workflow=bem-lambdas.yml --limit=3
gh run list --workflow=bem-step-functions.yml --limit=3

# Watch a specific run
gh run watch <run-id>
```

**Note:** The integration environment uses the QA JAR. When testing changes in integration:
1. Deploy JAR to `qa`
2. Deploy lambdas to `integration`
3. Deploy step functions to `integration`

### Working with Step Functions

Step functions are defined in `workflows/step-functions/*/statemachine/*.asl.json` and use **JSONata query language** (not JSONPath).

**Key step functions:**
- `batch-audience-delivery-daily-pipeline` - Entry point triggered daily by EventBridge
- `batch-audience-delivery-processing` - Main pipeline orchestrating BEM → Stitch → Formatter
- `bem-pipeline` - EMR cluster creation → BEM job execution → cluster termination
- `batch-stitch-pipeline` - Parallel execution of stitch jobs per vendor
- `batch-delivery-formatter-pipeline` - Formatting for vendor-specific requirements

## Key Concepts

### Expression Evaluation (BEM)

The BEM job (`BatchExpressionModelJob.scala`) evaluates audience expressions against SuperTagAv data:

1. **Input:**
   - `surveyexpressions.csv` - List of expressions to evaluate (e.g., `NOT (E999999998)`)
   - SuperTagAv parquet data - User behavioral data with attribute bitmaps (evmap, avmap, cvmap)
   - Models CSV - Expression evaluation models

2. **Processing:**
   - Reads SuperTagAv data (~2TB) with optimized schema (only needed bitmap columns)
   - Evaluates each expression against each RCID's attribute bitmaps
   - Uses `AudienceEvaluator` from `das-expression` library
   - Outputs matches partitioned by `exprHash`

3. **Output:** Parquet files with schema `(rcid, exprHash, score)` partitioned by `exprHash=<hash>`

### Identity Stitching

The Stitch job (`TableStitch.scala`) joins BEM output with vendor identity tables:

1. **Input:**
   - BEM cache (evaluated expressions with RCIDs)
   - Vendor stitch tables (e.g., `tmid`, `hemsha2`, `viant_id` columns)
   - Audience metadata (expression hash to audience ID mappings)

2. **Processing:**
   - Joins BEM cache with stitch table on `rcid`
   - Maps `exprHash` to `audienceIds` and explodes
   - Filters empty vendor keys

3. **Output:** `(akey, vkey, score)` where `akey` = audience key, `vkey` = vendor key (e.g., cookie, email hash)

### Delivery Configuration

The `batch-audience-delivery-processing-config` Lambda generates pipeline configuration:

- Queries RTP database for delivery schedules (via `RTP_PSQL_CONNECTION_STRING`)
- Determines which vendors need delivery based on schedule windows
- Generates S3 paths for tagav data, cookiejar, expressions, and output locations
- Sets EMR cluster configurations (instance types, counts, etc.)

**Important fields:**
- `delivery_type` - "daily", "training", "custom"
- `tagav_date_folder` - Date folder for SuperTagAv input (format: YYYYMMDD)
- `cookie_jar_date` - Date folder for cookiejar data
- `start_time_of_delivery` / `end_time_of_delivery` - Time window for vendor deliveries (epoch milliseconds)

### Formatter (CDP-118857)

The Formatter Spark job formats stitched data per vendor. Key recent changes:

**Output partition layout (changed in CDP-118857):**
- Old: `<prefix>/date=*/method=av/vendor=*/akey=*/`
- **New:** `<prefix>/date=*/vendor=*/method=*/akey=*/`

Any consumer that concatenates a `source_prefix` must use the new `vendor=*/method=*/` order. Affected repos: `batch-audience-delivery-syndication`, `batch-audience-delivery-experian-syndication`, `batch-audience-delivery-yahoo-syndication`, `batch-expression-modeling` (custom-delivery ASL).

**Metrics lambda (CDP-118857):** After EMR cluster termination, the formatter pipeline invokes `batch-delivery-formatter-publish-metrics`. Metrics are written to InfluxDB:
- `com.resonate.delivery.format.count` — per-audience rows
- `com.resonate.delivery.format.aggregate` — aggregate rows per vendor/method/delivery_type

**File extension caveat:** Spark's `partitionBy("method")` changed codec suffix separators from `-c000.csv.gz` to `.c000.csv.gz`. Publisher lambdas (`experian-publish-data-files`, `openx-publish-data-files`, etc.) must hardcode `.csv.gz` as the output extension rather than parsing it from the source filename.

**`delta_with_full_fallback` fix (CDP-118857):** `_annotate_previous_date_partitions` now accepts both `"delta"` and `"delta_with_full_fallback"` refresh types (mirrors Scala's `RefreshType.isDelta`). Meta deliveries use `delta_with_full_fallback`.

### Batch Stitch Pipeline (CDP-118972)

`AddJobFlowSteps` calls are staggered via index-based `Wait` states (iteration N waits N seconds before submission). This prevents AWS EMR `AddJobFlowSteps` throttling when N stitch steps are submitted in parallel.

The non-functional `EMR.ThrottlingException` retry was removed — the actual throttle error is `EMR.AmazonElasticMapReduceException`, not `EMR.ThrottlingException`.

### Prior-Stitch Partition Pruning (CDP-118857)

`prunePartitions` takes `dates: Seq[String]` and applies `col("date").isin(dates: _*)` as a static Catalyst filter. This prevents OOM on meta deliveries with `delta_with_full_fallback` by limiting the prior-stitch scan to only the dates referenced by current deliveries.

## AWS Step Functions: Converting JSONPath to JSONata

### Top-Level Changes

Add `"QueryLanguage": "JSONata"` at the state machine or state level.

### Field Replacements

JSONPath's five fields are replaced with two in JSONata:
- **Arguments** - Replaces `Parameters` for sending data to integrated actions
- **Output** - Replaces `ResultPath`, `ResultSelector`, and `OutputPath` for transforming state output

### JSONPath Expression Syntax

- **JSONPath**: `"field.$": "$.path.to.value"`
- **JSONata**: `"field": "{% $states.input.path.to.value %}"`

Remove the `.$` suffix and wrap expressions in `{% %}` delimiters.

### ResultPath Conversions

**Merge result into input at a path:**
```json
// JSONPath
"ResultPath": "$.ClusterCreationResult"

// JSONata
"Output": "{% $merge([$states.input, {'ClusterCreationResult': $states.result}]) %}"
```

**Discard result, pass through input only:**
```json
// JSONPath
"ResultPath": null

// JSONata
// Simply omit the Output field - input passes through by default
```

**Replace entire input with result (default):**
```json
// JSONPath
"ResultPath": "$"  // or omit ResultPath

// JSONata
// Omit Output field, or use:
"Output": "{% $states.result %}"
```

### OutputPath Conversions

**Extract specific field from result:**
```json
// JSONPath
"OutputPath": "$.Payload"

// JSONata
"Output": "{% $states.result.Payload %}"
```

### Catch Block Error Handling

**Merge error into input:**
```json
// JSONPath
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "ResultPath": "$.Error",
    "Next": "HandleError"
  }
]

// JSONata
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Output": "{% $merge([$states.input, {'Error': $states.errorOutput}]) %}",
    "Next": "HandleError"
  }
]
```

Note: `$states.errorOutput` is only available in Catch blocks.

### Map State Conversions

**ItemsPath:**
```json
// JSONPath
"ItemsPath": "$.step_configs"

// JSONata
"Items": "{% $states.input.step_configs %}"
```

**ItemSelector:**
```json
// JSONPath
"ItemSelector": {
  "step_config.$": "$$.Map.Item.Value",
  "clusterId.$": "$.ClusterCreationResult.ClusterId"
}

// JSONata
"ItemSelector": {
  "step_config": "{% $states.context.Map.Item.Value %}",
  "clusterId": "{% $states.input.ClusterCreationResult.ClusterId %}"
}
```

**IMPORTANT:** Use `$states.context.Map.Item.Value` to access the current item in a Map state. There is **NO** `$states.item` variable - this is a common mistake that will cause "Field '$states.item' does not exist" errors.

**⚠️ CRITICAL: Map States Must Preserve Input**

By default, a Map state outputs an **array of results** from processing each item, which **loses the original input fields**. If subsequent states need access to fields from the original input (like a ClusterId from cluster creation), you MUST add an Output field to preserve the input.

```json
// WRONG - This loses ClusterCreationResult from input
"Step Map": {
  "Type": "Map",
  "ItemSelector": {
    "ClusterId": "{% $states.input.ClusterCreationResult.ClusterId %}"
  },
  "Items": "{% $states.input.step_configs %}",
  "Next": "TerminateCluster"  // ❌ Will fail - ClusterCreationResult is lost!
}

// CORRECT - Preserve input for next state
"Step Map": {
  "Type": "Map",
  "ItemSelector": {
    "ClusterId": "{% $states.input.ClusterCreationResult.ClusterId %}"
  },
  "Items": "{% $states.input.step_configs %}",
  "Output": "{% $states.input %}",  // ✅ Preserves all input fields
  "Next": "TerminateCluster"
}
```

**Common Error:**
```
The JSONata expression '$states.input.ClusterCreationResult.ClusterId' specified for the field 'Arguments/ClusterId' returned nothing (undefined).
```

This error means the Map state didn't preserve the input, so subsequent states can't access fields that were in the original input.

### Array Construction

**Intrinsic function to array literal:**
```json
// JSONPath
"Args.$": "States.Array('cmd', '--flag', $.input.value, '--opt', $.input.other)"

// JSONata - Use regular JSON array with individual JSONata expressions
"Args": [
  "cmd",
  "--flag",
  "{% $states.input.value %}",
  "--opt",
  "{% $states.input.other %}"
]
```

### Reserved Variables in JSONata

- `$states.input` - Original input to the current state
- `$states.result` - Result from API/sub-workflow (Task, Parallel, Map states)
- `$states.errorOutput` - Error output (only in Catch blocks)
- `$states.context` - Execution metadata (StartTime, task token, Map.Item.Value, etc.)
  - `$states.context.Map.Item.Value` - Current item in Map state iteration
  - `$states.context.Map.Item.Index` - Current iteration index in Map state

### ⚠️ Common Pitfall: $states.item Does NOT Exist

**Error:** `Field '$states.item' does not exist`

**WRONG:**
```json
"ItemSelector": {
  "item": "{% $states.item %}"  // ❌ This will fail!
}
```

**CORRECT:**
```json
"ItemSelector": {
  "item": "{% $states.context.Map.Item.Value %}"  // ✅ Use this instead
}
```

Many online examples and AI suggestions incorrectly reference `$states.item`, but this variable does not exist in AWS Step Functions JSONata. Always use `$states.context.Map.Item.Value` to access the current Map iteration item.

### Key Differences from JSONPath

1. **No `.$` suffix** - Regular field names with JSONata expressions in `{% %}`
2. **No Path fields** - InputPath, ResultPath, OutputPath are replaced by Arguments and Output
3. **Assign vs Output** - `Assign` creates variables accessible in all future states; `Output` only affects the immediate next state
4. **Parallel processing** - Assign and Output are processed in parallel; variable assignments don't affect Output

### Common Patterns

**Pass through input unchanged:**
- Omit the Output field entirely, OR explicitly use `"Output": "{% $states.input %}"`

**IMPORTANT:** Any Task state that needs to pass data to subsequent states should preserve the input explicitly. Without an Output field, the Task's result replaces the entire input, losing all previous data.

**Common scenarios requiring input preservation:**

1. **After Catch blocks** - When error info was merged into input
2. **Cleanup/termination tasks** - When subsequent states need original context (e.g., Lambda functions that publish metrics need delivery_config)
3. **Between pipeline stages** - When chaining multiple operations that all need access to the original request

```json
// Example 1: Catch block merges error into input
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Output": "{% $merge([$states.input, {'Error': $states.errorOutput}]) %}",
    "Next": "CleanupTask"
  }
]

// The CleanupTask MUST preserve input if Error is needed later
"CleanupTask": {
  "Type": "Task",
  "Resource": "arn:aws:states:::elasticmapreduce:terminateCluster",
  "Arguments": { "ClusterId": "{% $states.input.ClusterCreationResult.ClusterId %}" },
  "Output": "{% $states.input %}",  // ✅ Preserves Error field
  "Next": "NotifyFailure"
}

// Example 2: EMR termination before Lambda that needs delivery config
"EMR TerminateCluster Success": {
  "Type": "Task",
  "Resource": "arn:aws:states:::elasticmapreduce:terminateCluster",
  "Arguments": { "ClusterId": "{% $states.input.ClusterCreationResult.ClusterId %}" },
  "Output": "{% $states.input %}",  // ✅ Lambda needs delivery_config from input
  "Next": "Publish Metrics"
}
```

**Add result as new field while preserving input:**
```jsonata
"Output": "{% $merge([$states.input, {'resultField': $states.result}]) %}"
```

**Transform and extract specific data:**
```jsonata
"Output": {
  "field1": "{% $states.input.someValue %}",
  "field2": "{% $states.result.someOtherValue %}"
}
```

**Create variables for use in later states:**
```jsonata
"Assign": {
  "myVariable": "{% $states.input.someValue %}"
}
// Later states can reference {% $myVariable %}
```

### Reference Documentation

- [AWS Step Functions - Transforming data with JSONata](https://docs.aws.amazon.com/step-functions/latest/dg/transforming-data.html)
- [AWS Step Functions - Passing data between states with variables](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-variables.html)
