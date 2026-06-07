# identity-graph

## Project Purpose and Architecture Overview

This repository hosts the **PRISM Identity Graph** — Resonate's person identity resolution system. It maps customer-supplied identifiers (HEM, MAID, ZIP11, name+address, etc.) to a canonical `person_id`, enabling cross-channel identity stitching at scale.

The core deliverable is **`prism_dbt`** — a Snowflake-native dbt package deployed via `CREATE DBT PROJECT` / `EXECUTE DBT PROJECT`. Consumers invoke it from Step Functions (via Lambda) or directly from Snowflake SQL.

**Top-level layout:**

```
dbt/
  prism_dbt/                  # The Snowflake dbt package (v1.0+)
    macros/                   # Core primitives (waterfall_match, identifier_expand, etc.)
    models/
      service/                # Lambda-invokable service models
        waterfall.sql         # Main integration point for consumers
    tests/
      unit/                   # Offline dbt unit tests (no Snowflake needed)
      *.sql                   # Singular tests (require live Snowflake mock graph)
    dbt_project.yml
    profiles.yml.example      # Template for local dev
snowflake/
  function/
    name_address_hash/        # NAME_ADDRESS_HASH UDF (SHA-256 PII normalization)
  stored_procedures/
    prism_lookup/             # PRISM_LOOKUP stored procedure
    name_address_lookup/      # NAME_ADDRESS_LOOKUP stored procedure
.github/
  workflows/
    snowflake_dbt.yml         # Deploys prism_dbt package to Snowflake
    snowflake_stored_procedure.yml  # Deploys PRISM_LOOKUP / NAME_ADDRESS_LOOKUP SPs
    snowflake_function.yml    # Deploys NAME_ADDRESS_HASH UDF
```

**Production JAR path (Scala identity graph):**
`s3://resonate-core-applications/identity-graph/jars/identity-graph-latest.jar`

---

## Core Primitives

| Primitive | Direction | Purpose |
|---|---|---|
| `prism_dbt.waterfall_match` | identifiers → ONE person_id per row | Resolve mixed-identifier input to a canonical `person_id` via priority-ordered waterfall (first-match-wins) |
| `prism_dbt.identifier_expand` | person_id → identifiers of ONE type | Fan out `person_id`s to deliverable identifiers (HEM, MAID, TTD, etc.) |
| `prism_dbt.persons_project` | person_id → persons attributes | Project `persons` columns (name, address, lalvoterid, …) onto `person_id`s |
| `prism_dbt.lookup` | one identifier → one person + linked IDs | Single-row debug lookup; also deployed as `PRISM_LOOKUP` stored procedure |
| `NAME_ADDRESS_LOOKUP` (SP) | PII tuple → one person | Resolves raw `(first_name, last_name, address_line, zip)` to a person record |

**Picking the right primitive:**
- *"I have customer rows with mixed identifiers, give me one person_id each"* → `waterfall_match`
- *"I have person_ids, give me their RAMP_IDs (or TTDs, or CTV_IDs)"* → `identifier_expand` (one call per type)
- *"I have person_ids, give me name + address"* → `persons_project`
- *"I have one identifier and want to debug who it resolves to"* → `lookup` macro or `PRISM_LOOKUP` SP
- *"I have raw name+address and want the person_id"* → `NAME_ADDRESS_LOOKUP` SP

---

## Common Development Tasks

### Local Development (prism_dbt)

```bash
cd dbt/prism_dbt

# Install dependencies
dbt deps

# Run against mock graph (requires ~/.dbt/profiles.yml — copy from profiles.yml.example)
dbt seed --profiles-dir <your-profiles-dir>
dbt run --select waterfall --profiles-dir <your-profiles-dir>
dbt test --profiles-dir <your-profiles-dir>
```

**Two profile files — don't confuse them:**
- `profiles.yml.example` → copy to `~/.dbt/profiles.yml` for local `dbt run` / `dbt test` (uses real SSO auth)
- `snowflake_profiles/profiles.yml` → stub for `snow dbt deploy` only (account/user = `'_'`). **Do not edit.**

**Mock tables** live in `RESONATE.PRISM.*` on `resonate-dev`, provisioned from `.context/cdp-119018-mock-tables.sql`.

### Running Tests

```bash
dbt test                              # All tests (25 total: 16 unit + 9 singular)
dbt test --select test_type:unit      # Unit tests only (offline — no Snowflake needed)
dbt test --select test_type:data      # Singular tests only (require live Snowflake)
```

**Unit tests** (`tests/unit/*.yml`) use dbt's `unit_tests:` framework — mocked inline, no Snowflake required. Fast and deterministic.

**Singular tests** (`tests/test_*.sql`) run against the live `RESONATE.PRISM.*` mock graph on `resonate-dev`. Used for scenarios that can't be mocked (UDF calls, stored procedures, `input_column_aliases` with custom column names).

### Deploying to Snowflake

All deployments go through GitHub Actions:

| Workflow | What it does |
|---|---|
| `snowflake_dbt.yml` | Uploads `prism_dbt` via `snow dbt deploy`. Runs `dbt parse` first as fast-fail gate. |
| `snowflake_stored_procedure.yml` | Deploys `PRISM_LOOKUP` or `NAME_ADDRESS_LOOKUP` SPs with placeholder substitution. |
| `snowflake_function.yml` | Deploys the `NAME_ADDRESS_HASH` UDF with placeholder substitution. |

---

## Key Concepts

### `waterfall_match` Step Config

Caller-supplied list of step dicts; first match wins per input row:

| Key | Required | Description |
|---|---|---|
| `identifier_type` | yes | PRISM identifier type (`HEM_SHA256`, `MAID`, `ZIP11`, …) |
| `stitch_label` | yes | String emitted in `prism_matched_step` on match |
| `input_column` | no | Caller input column. Defaults: `HEM_TYPE`-gated for HEM variants; else `identifier_type` as column name |
| `input_type_column`, `input_type_value` | no | Gate the step on a sibling column value |
| `link_basis` | no | Crosswalk filter (`DIRECT`, `VIA_PII_MATCH`, `VIA_HEM`, etc.) |
| `output_value` | no | `'rid'` (default) or `'identifier_value'` |

**ZIP11 is special:** It lives as a column on `persons`, NOT in `identifier_index`. `waterfall_match` routes ZIP11 lookups directly to `persons.zip11` automatically — no special caller config needed.

### Default Waterfall Priority

`prism_default_waterfall` (PRISM's recommended order, high-precision first):
HEM_MD5 → HEM_SHA1 → HEM_SHA256 → MAID → RAMP_ID → TTD → CTV_ID → RCID → L2_VOTER_ID → L2_CONSUMER_ID → ZIP11

**Intentionally excluded from default** (opt-in only): `IP`, `PHONE_CELL`, `PHONE_LANDLINE`, `NONID`, `HOUSEHOLD_ID`, `L2_FAMILY_ID`, `EXPERIAN_*`, `NAME_ADDRESS`.

**`RID` is not published by PRISM** — it's a Resonate-internal bridge owned by Append's downstream pipeline.

### NAME_ADDRESS Normalization (v1.0)

UDF normalization recipe: `LOWER + TRIM + strip non-alphanumeric + LPAD zip + pipe-concat + SHA-256`.

To enable NAME_ADDRESS in a waterfall:
```sql
SELECT
  *,
  {{ prism_dbt.name_address_hash('first_name', 'last_name', 'address_line', 'zip') }} AS NAME_ADDRESS
FROM normalized_input_raw
```
Then add `{identifier_type: NAME_ADDRESS, stitch_label: name_address}` to `waterfall_order`.

v1.0 does **not** do canonical-first-name lookup (Bill ≠ William) — v1.1 adds it.

### `waterfall_match` Output Schema

One row per input row (input pass-through + PRISM columns):
```
prism_matched_step               -- stitch_label of winning step (NULL if no match)
prism_matched_priority           -- 1-based priority of winning step
prism_matched_identifier_type
prism_matched_identifier_value
prism_matched_person_id          -- THE key output
prism_matched_output_mode
prism_consumer
prism_called_at
```

Unmatched rows are preserved with NULL `prism_matched_*` columns.

### Invoking from a Step Function / Lambda

```python
import json, snowflake.connector, os

def handler(event, context):
    args = {
        "input_relation":         event["input_relation"],
        "consumer":               event["consumer"],
        "job_run_id":             event["job_run_id"],
        "waterfall_order":        event["waterfall_order"],
        "output_database":        event["output_database"],
        "output_schema":          event["output_schema"],
        "waterfall_output_alias": event["output_alias"],
    }
    sql = f"""
        EXECUTE DBT PROJECT RESONATE.PRISM.prism_dbt
          ARGS = $$run --select waterfall --vars '{json.dumps(args)}'$$
    """
    with snowflake.connector.connect(
        account=os.environ["SF_ACCOUNT"],
        user=os.environ["SF_USER"],
        password=os.environ["SF_PASSWORD"],
        warehouse="RESONATE_X3LARGE_WH",
        role="ACCOUNTADMIN",
    ) as conn:
        with conn.cursor() as cur:
            cur.execute(sql)
```

---

## Project-Specific Rules and Gotchas

- **`CROSSWALK_LATEST` is NOT used by v1.0 macros.** The prod crosswalk has a 2-column schema missing metadata needed for confidence filtering. v1.0 uses `PERSONS_LATEST` and `IDENTIFIER_INDEX_LATEST` only.
- **`identifier_rank = 1` filter is enforced.** `waterfall_match` only considers canonical-person rows from `identifier_index`. Rank-2+ rows (fan-out) are excluded from match. Use `identifier_expand` if you want all linked identifiers.
- **Deterministic tiebreak always applies.** Even for fan-out identifier types (RCID, TTD, RAMP_ID), `waterfall_match` picks one person deterministically. Order: priority → confidence DESC → source_count DESC → max_effective_weight DESC → person_id ASC.
- **`min_confidence` default is 0.7.** Override to `0.0` for Append parity baseline. This filters on `identifier_index.confidence`.
- **Service role is `ACCOUNTADMIN`.** Cross-DB CTAS into any consumer DB works without per-client grants. Do not downgrade this role without verifying all consumer DB writes still succeed.
- **`prism_database` / `prism_schema` vars:** In prod, set `prism_database=PRISM`, `prism_schema=PUBLIC`. In dev/non-prod, defaults are `RESONATE` / `PRISM`.

---

## Recent Changes (as of June 2026)

### prism_dbt v1.0 Release (CDP-119018, June 2026)

The `prism_dbt` package reached v1.0. Key additions:
- **`waterfall_match` macro** with full ZIP11 routing (ZIP11 now resolved via `persons.zip11` column, not `identifier_index`)
- **`NAME_ADDRESS_HASH` UDF** deployed to Snowflake; callable via `prism_dbt.name_address_hash()` macro
- **`NAME_ADDRESS_LOOKUP` stored procedure** for single-row PII-based lookups
- **CI workflows** added for automated dbt package deployment, stored procedure deployment, and UDF deployment
- **Schema-level data tests** for all three service models
- **`prism_default_waterfall`**: `IP` and `ZIP11` routing clarified — `MAID` and `IP` are single-type (not split by platform/version); ZIP11 is excluded from default (household-precision)
- **`PRISM` database**: Snowflake DB choices in workflows no longer include a separate `PRISM` DB option — everything lives in `RESONATE.PRISM.*`

### ZIP11 Routing Fix

ZIP11 was previously included in `identifier_index` lookups. It's now correctly routed to `persons.zip11` column directly (see `_internal/lookup_strategy.sql`). If you see unexpected ZIP11 behavior, check that you're on v1.0 of the package.
