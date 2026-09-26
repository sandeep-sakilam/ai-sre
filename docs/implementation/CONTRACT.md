# API contract

The single source of truth for the HTTP API between the frontend and the backend. Only the FE session edits
this file, and only between phases (see [README.md](README.md)). The Pydantic models in `src/api/schemas.py`
must match it.

- Base URL in local development: `http://localhost:8000`
- JSON bodies use `snake_case` keys. Missing values are JSON `null`, never `NaN`.
- CORS allows the origins in the `CORS_ORIGINS` env var (the Vite dev server by default) with methods
  `GET`, `POST` and `OPTIONS`.

| Method | Path | Since | Purpose |
|---|---|---|---|
| `GET` | `/health` | existing | Is the backend up, and is an LLM configured? |
| `GET` | `/api/demo/scenario` | existing | Demo run: execution summary and validation results |
| `GET` | `/api/demo/run` | existing | Demo run: pipeline run record and raw rows |
| `POST` | `/api/investigate` | existing | AI investigation |
| `GET` | `/api/targets` | **Phase 2** | List the target tables |
| `GET` | `/api/targets/{target_id}` | **Phase 2** | One target: schema, DDL, row count, recent rows, demo files |
| `GET` | `/api/targets/{target_id}/samples/{file_name}` | **Phase 2** | Download a demo CSV |
| `POST` | `/api/targets/{target_id}/reset` | **Phase 2** | Restore the target table's seed data |
| `POST` | `/api/targets/{target_id}/runs` | **Phase 2** | Upload one CSV and run the load pipeline |
| `GET` | `/api/targets/{target_id}/runs` | **Phase 2** | Run history for a target, newest first |
| `GET` | `/api/runs/{run_id}` | **Phase 2** | One run, including its investigation if there is one |
| `POST` | `/api/runs/{run_id}/investigate` | **Phase 2** | AI investigation of a failed run |

The Phase 1 endpoints (a two-file `POST /api/runs`) were cancelled and are not part of the contract.

---

## Existing endpoints (unchanged)

These are implemented and tested. Don't change them.

### `GET /health`
```json
{ "status": "ok", "llm_configured": true, "details": { "provider": "AnthropicProvider", "model": "claude-sonnet-5" } }
```
Without an API key: `"llm_configured": false, "details": { "error": "No API key set for provider 'anthropic'. Set LLM_API_KEY." }`.

### `GET /api/demo/scenario`
Model: `DemoScenarioResponse`.
```json
{
  "pipeline_name": "Customer Nightly Sync",
  "pipeline_description": "Nightly customer sync pipeline. ...",
  "status": "FAILED",
  "execution_summary": {
    "source_records": 20, "target_records": 23, "run_id": "run-2024-04-05-0200",
    "reported_run_status": "SUCCESS", "started_at": "2024-04-05T02:00:00Z", "finished_at": "2024-04-05T02:03:41Z"
  },
  "validation_summary": { "total_checks": 6, "passed_checks": 0, "failed_checks": 6 },
  "validation_results": [
    {
      "name": "duplicate_customer_id", "status": "FAILED", "severity": "HIGH",
      "summary": "5 customer ID(s) duplicated in target (5 extra row(s))",
      "metrics": { "affected_records": 10, "duplicate_ids": 5, "extra_rows": 5 },
      "evidence": ["customer_id 1012 appears 2 times in target", "..."]
    }
  ]
}
```
`status` is `PASSED` or `FAILED`. `severity` is `HIGH`, `MEDIUM` or `LOW`. `metrics` values are numbers
or `{value: count}` maps. `evidence` has at most 5 lines plus an optional `"... and N more"` line.

### `GET /api/demo/run`
Model: `DemoRunResponse`.
```json
{
  "pipeline_run": {
    "run_id": "run-2024-04-05-0200", "pipeline": "customer_nightly_sync", "status": "SUCCESS",
    "started_at": "2024-04-05T02:00:00Z", "finished_at": "2024-04-05T02:03:41Z",
    "config": { "batch_size": 5 },
    "steps": [ { "name": "extract", "status": "SUCCESS", "rows_in": 20, "rows_out": 20 } ],
    "logs": [ "02:00:00 INFO  extract: read 20 rows from source_customers.csv" ]
  },
  "source": [ { "customer_id": "1001", "name": "Alice Carter", "email": "Alice.Carter@example.com", "country": "US", "signup_date": "2024-01-05", "status": "active" } ],
  "target": [ { "customer_id": null, "name": "Tara Novak", "...": "..." } ]
}
```

### `POST /api/investigate`
Model: `InvestigateRequest`, then `InvestigateResponse`. The UI sends:
```json
{ "pipeline_name": "Customer Nightly Sync", "pipeline_description": "...", "execution_summary": { "...": "..." }, "use_demo_data": true }
```
The response has `summary`, `observed_facts`, `hypotheses` (`hypothesis`, `status`
`SUPPORTED|REJECTED|INCONCLUSIVE`, `evidence`, `reasoning`), `root_cause_status` (`IDENTIFIED|INCONCLUSIVE`),
`root_cause`, `root_cause_evidence`, `root_cause_reasoning`, `evidence` (alias of `observed_facts`),
`recommended_fix`, `regression_test` (string), `investigation_trace` (`stage`, `description`) and
`evidence_warnings`. Evidence lines look like `"[validation.duplicate_customer_id] text"`.

Errors use `{"detail": "<message>"}`: `503` (no LLM configured), `504` (LLM timeout), `502` (provider error
or invalid LLM output), `500` (unexpected). Request validation errors are FastAPI's default `422`.

---

## Phase 2: the target load flow

### Conventions for every Phase 2 endpoint

- IDs: `target_id` is a lowercase slug (`customers`). `run_id` is `run_` followed by 12 lowercase hex characters.
- Times are UTC, ISO 8601, second precision, with a `Z` suffix: `"2026-09-26T10:15:00Z"`.
- A row number is 1-based and counts data rows only: `row_number: 1` is the first row after the header.
- Every error uses the structured form:
  ```json
  { "detail": { "code": "target_not_found", "message": "Target 'orders' does not exist.", "field": null } }
  ```
  `message` is a plain sentence the UI shows as-is. `field` names the request field at fault, or is `null`.
  FastAPI's default `422` (a list in `detail`) can still occur for a malformed request; the frontend handles it.

| Status | `code` | When |
|---|---|---|
| `404` | `target_not_found` | Unknown `target_id` |
| `404` | `run_not_found` | Unknown `run_id`, including after a backend restart |
| `404` | `sample_not_found` | Unknown demo file name |
| `413` | `file_too_large` | The CSV is over 10 MB (10,485,760 bytes) |
| `422` | `invalid_encoding` | The CSV isn't UTF-8 (a BOM is allowed) |
| `422` | `invalid_csv` | The CSV is empty, has a header but no rows, has an empty or duplicate header name, or can't be parsed |
| `422` | `too_many_rows` | More than 50,000 data rows |
| `422` | `invalid_previous_run` | `previous_run_id` is unknown or belongs to another target |
| `409` | `run_not_failed` | Investigating a run whose status is `LOADED` |
| `503` | `llm_not_configured` | No LLM API key |
| `504` | `llm_timeout` | The LLM didn't answer in time |
| `502` | `llm_error` | The provider failed, or returned output that is invalid even after one retry |
| `500` | `internal_error` | Anything unexpected. Never include a stack trace |

A file that can't be read as a CSV at all (the `413` and `422` codes above) is rejected with **no run created**.
Everything else, including missing columns, creates a run that ends `FAILED` with evidence.

### Data shapes

**`ColumnSpec`** (one column of a target table):
```json
{
  "name": "email",
  "type": "TEXT",
  "nullable": false,
  "primary_key": false,
  "description": "Customer email address",
  "rules": [ { "kind": "email", "value": null, "description": "Must be a valid email address" } ]
}
```
- `type` is `INTEGER`, `TEXT` or `DATE`. A `DATE` is stored as `TEXT` in `YYYY-MM-DD` form.
- `rules[].kind` is one of:
  - `email`: `value` is `null`.
  - `pattern`: `value` is a regular expression, for example `^[A-Z]{2}$`.
  - `allowed_values`: `value` is a list of strings, compared case-sensitively.

**`CheckResult`** has exactly the same shape as in `/api/demo/scenario`: `name`, `status` (`PASSED|FAILED`),
`severity` (`HIGH|MEDIUM|LOW`), `summary`, `metrics`, `evidence` (at most 5 lines plus an optional
`"... and N more"`).

**`PipelineStep`**: `{ "name", "status", "rows_in", "rows_out", "duration_ms", "message" }`.
- `status` is `SUCCESS`, `FAILED` or `SKIPPED`.
- `rows_in` and `rows_out` are integers, or `null` for a skipped step.
- `message` is one sentence.

**`RowIssue`**: `{ "row_number", "column", "value", "check", "message" }`.
- `value` is the raw string from the file, or `null` if the cell is empty.
- `check` is the `name` of the `CheckResult` that found the issue.

### Validation checks

The checks are generated from the target's schema. Check names are stable: the frontend shows them generically,
and the AI cites them as `validation.<name with dots replaced by underscores>`.

| Check `name` | Generated for | Fails when | Severity |
|---|---|---|---|
| `schema.missing_columns` | every run | A target column is not in the file header | HIGH |
| `schema.unexpected_columns` | every run | The file has a column the target doesn't | MEDIUM |
| `not_null.<column>` | each non-nullable column | A cell is empty | HIGH |
| `type.<column>` | each `INTEGER` or `DATE` column | A value doesn't parse (`INTEGER`: optional sign and digits only; `DATE`: a real `YYYY-MM-DD` date) | HIGH |
| `format.<column>` | each `email` or `pattern` rule | A value doesn't match | MEDIUM |
| `allowed_values.<column>` | each `allowed_values` rule | A value isn't in the list | MEDIUM |
| `unique_in_file.<column>` | the primary key | The same key appears more than once in the file | HIGH |
| `key_conflict.<column>` | the primary key | The key already exists in the target table | HIGH |

- Row-level checks skip empty cells, except `not_null`.
- If `schema.missing_columns` fails, the row-level checks for the missing columns are still listed, with status
  `PASSED` and summary `"Not run: column missing from file"`. The list of checks is therefore the same for
  every run of a target.
- Order: the two schema checks first, then per column in schema order: `not_null`, `type`, `format`,
  `allowed_values`, `unique_in_file`, `key_conflict`.
- Every check lists `metrics.affected_rows` (an integer).

### The pipeline

Every run executes these steps in this order. After a step fails, the later steps are `SKIPPED`.

| Step `name` | What it does | Fails when |
|---|---|---|
| `parse` | Reads the CSV (every value as a string; an empty cell is `null`; `NA`, `null` and `None` stay text) | never. Unreadable files are rejected before a run exists |
| `validate_schema` | Compares the header with the target's columns | `schema.*` fails |
| `validate_rows` | Runs the row-level checks | any row-level check fails |
| `load` | Converts types and inserts every row into the SQLite table in one transaction | the database rejects the insert. The transaction is rolled back and nothing is loaded |

`logs` is a list of lines in the form `"HH:MM:SS LEVEL message"` with `LEVEL` one of `INFO`, `WARN` or `ERROR`,
for example `"10:15:02 ERROR validate_rows: 7 rows failed 5 checks"`. These are written by the pipeline as it
runs; they are not canned text.

### `GET /api/targets`

`200`:
```json
[
  { "target_id": "customers", "name": "Customers", "table_name": "customers",
    "description": "Warehouse table of active CRM customers, loaded nightly from the CRM export.",
    "row_count": 12, "column_count": 6 }
]
```

### `GET /api/targets/{target_id}`

`200` returns a `TargetDetail`:
```json
{
  "target_id": "customers",
  "name": "Customers",
  "table_name": "customers",
  "description": "Warehouse table of active CRM customers, loaded nightly from the CRM export.",
  "ddl": "CREATE TABLE customers (\n  customer_id INTEGER PRIMARY KEY,\n  name TEXT NOT NULL,\n  email TEXT NOT NULL,\n  country TEXT NOT NULL,\n  signup_date TEXT NOT NULL,\n  status TEXT NOT NULL\n);",
  "columns": [
    { "name": "customer_id", "type": "INTEGER", "nullable": false, "primary_key": true, "description": "CRM customer number", "rules": [] },
    { "name": "name", "type": "TEXT", "nullable": false, "primary_key": false, "description": "Full name", "rules": [] },
    { "name": "email", "type": "TEXT", "nullable": false, "primary_key": false, "description": "Customer email address",
      "rules": [ { "kind": "email", "value": null, "description": "Must be a valid email address" } ] },
    { "name": "country", "type": "TEXT", "nullable": false, "primary_key": false, "description": "ISO 3166 country code",
      "rules": [ { "kind": "pattern", "value": "^[A-Z]{2}$", "description": "Two uppercase letters, e.g. US" } ] },
    { "name": "signup_date", "type": "DATE", "nullable": false, "primary_key": false, "description": "Date the customer signed up", "rules": [] },
    { "name": "status", "type": "TEXT", "nullable": false, "primary_key": false, "description": "Lifecycle status",
      "rules": [ { "kind": "allowed_values", "value": ["active", "inactive", "churned"], "description": "One of active, inactive, churned" } ] }
  ],
  "row_count": 12,
  "recent_rows": [
    { "customer_id": 1012, "name": "Laura Chen", "email": "laura.chen@example.com", "country": "US", "signup_date": "2024-03-04", "status": "active" }
  ],
  "samples": [
    { "file_name": "customers_batch_2024-04-05.csv", "description": "Nightly CRM export with problems the load will reject.", "expected_status": "FAILED" },
    { "file_name": "customers_batch_2024-04-05_fixed.csv", "description": "The same export, corrected.", "expected_status": "LOADED" }
  ]
}
```
- `recent_rows` holds the last 20 rows in insertion order, newest first, with typed values (integers as JSON numbers).
- `404 target_not_found`.

### `GET /api/targets/{target_id}/samples/{file_name}`

`200` with `content-type: text/csv` and `content-disposition: attachment; filename="<file_name>"`.
`404 target_not_found` or `404 sample_not_found`.

### `POST /api/targets/{target_id}/reset`

Drops and recreates the table, then inserts the seed rows. Run history is kept.
`200` returns the `TargetDetail`. `404 target_not_found`.

### `POST /api/targets/{target_id}/runs`

`multipart/form-data`:

| Field | Required | Content |
|---|---|---|
| `file` | yes | The CSV to load: UTF-8, comma-separated, header row first |
| `previous_run_id` | no | The failed run this upload corrects. It links the attempts together |

The pipeline runs synchronously. `201 Created` returns a `Run`, whether it ends `LOADED` or `FAILED`.

A failed run:
```json
{
  "run_id": "run_3f9a1c2b7d4e",
  "target_id": "customers",
  "file_name": "customers_batch_2024-04-05.csv",
  "created_at": "2026-09-26T10:15:00Z",
  "status": "FAILED",
  "attempt": 1,
  "previous_run_id": null,
  "summary": {
    "rows_received": 10, "rows_valid": 3, "rows_rejected": 7, "rows_loaded": 0,
    "target_rows_before": 12, "target_rows_after": 12
  },
  "pipeline": {
    "steps": [
      { "name": "parse", "status": "SUCCESS", "rows_in": 10, "rows_out": 10, "duration_ms": 3, "message": "Read 10 rows and 6 columns." },
      { "name": "validate_schema", "status": "SUCCESS", "rows_in": 10, "rows_out": 10, "duration_ms": 1, "message": "All 6 target columns present; no unexpected columns." },
      { "name": "validate_rows", "status": "FAILED", "rows_in": 10, "rows_out": 3, "duration_ms": 9, "message": "7 of 10 rows failed 5 checks." },
      { "name": "load", "status": "SKIPPED", "rows_in": null, "rows_out": null, "duration_ms": 0, "message": "Skipped because validation failed; nothing was loaded." }
    ],
    "logs": [
      "10:15:00 INFO parse: read 10 rows from customers_batch_2024-04-05.csv",
      "10:15:00 INFO validate_schema: header matches target customers",
      "10:15:00 ERROR validate_rows: 7 rows failed 5 checks",
      "10:15:00 WARN load: skipped; 0 of 10 rows loaded"
    ]
  },
  "validation": {
    "total_checks": 15, "passed_checks": 10, "failed_checks": 5,
    "results": [
      { "name": "key_conflict.customer_id", "status": "FAILED", "severity": "HIGH",
        "summary": "1 row has a customer_id that already exists in customers",
        "metrics": { "affected_rows": 1 },
        "evidence": ["row 1: customer_id 1005 already exists in customers"] }
    ]
  },
  "row_issues": [
    { "row_number": 1, "column": "customer_id", "value": "1005", "check": "key_conflict.customer_id",
      "message": "customer_id 1005 already exists in customers" }
  ],
  "row_issues_total": 7,
  "investigation": null
}
```
- `validation.results` lists every check, including passed ones, in the order given above.
- `row_issues` lists at most 500 issues, sorted by `row_number` and then column order. `row_issues_total` is the
  full count. One row can have several issues.
- A `LOADED` run has `failed_checks: 0`, every step `SUCCESS`, `rows_loaded` equal to `rows_received`,
  `target_rows_after` equal to `target_rows_before + rows_loaded`, `row_issues: []`, and `investigation: null`.
- `attempt` is 1 without `previous_run_id`; otherwise it is the previous run's `attempt` plus 1.
- Errors: `404 target_not_found`, `413 file_too_large`, `422 invalid_encoding`, `invalid_csv`, `too_many_rows`,
  `invalid_previous_run`.

### `GET /api/targets/{target_id}/runs`

`200`, newest first:
```json
[
  { "run_id": "run_9b2e44c0a1f3", "file_name": "customers_batch_2024-04-05_fixed.csv", "created_at": "2026-09-26T10:21:40Z",
    "status": "LOADED", "attempt": 2, "previous_run_id": "run_3f9a1c2b7d4e", "rows_received": 8, "rows_loaded": 8 },
  { "run_id": "run_3f9a1c2b7d4e", "file_name": "customers_batch_2024-04-05.csv", "created_at": "2026-09-26T10:15:00Z",
    "status": "FAILED", "attempt": 1, "previous_run_id": null, "rows_received": 10, "rows_loaded": 0 }
]
```
`404 target_not_found`.

### `GET /api/runs/{run_id}`

`200` returns the `Run`. Its `investigation` is `null`, or the latest investigation result. `404 run_not_found`.

### `POST /api/runs/{run_id}/investigate`

No request body. The backend builds the evidence package from **this run only**:
- the target's schema and DDL
- the pipeline steps and logs
- every check result
- a profile of the uploaded file (row count, columns, empty-cell counts per column)
- the first 50 row issues

It then calls the existing investigation service (the same prompt, parsing, retry, citation check and trace).

Evidence IDs the AI can cite:
- `target.schema`
- `pipeline.run_log`
- `validation.summary`
- `validation.<check name with dots replaced by underscores>`
- `dataset.upload`
- `dataset.row_issues`

`200` returns exactly the existing `InvestigateResponse` shape (see `POST /api/investigate`). The result is also
stored on the run, replacing any earlier one, so `GET /api/runs/{run_id}` returns it after a page reload.

Errors: `404 run_not_found`, `409 run_not_failed`, `503 llm_not_configured`, `504 llm_timeout`, `502 llm_error`,
`500 internal_error`.

### The demo data (the backend creates it; the frontend relies on these facts)

**Seed rows:** 12 customers with `customer_id` 1001 to 1012. All valid.

**`customers_batch_2024-04-05.csv`:** 10 rows with the target's 6 columns in schema order. The rows have these
`customer_id` values, in order: `1005, 1013, 1014, 1017, 1017, CUST-1020, 1018, 1019, 1021, 1022`. The defects are
exactly these, and no others:

| Row | Defect | Failing check |
|---|---|---|
| 1 | `customer_id` 1005, which is already in the table | `key_conflict.customer_id` |
| 4 and 5 | `customer_id` 1017 twice; both rows are flagged | `unique_in_file.customer_id` |
| 6 | `customer_id` `CUST-1020` | `type.customer_id` |
| 7 | email `george.brown@example` | `format.email` |
| 8 | status `Active` | `allowed_values.status` |
| 9 | status `pending` | `allowed_values.status` |

That is 7 issues in 7 rows across 5 failed checks, with 3 valid rows (2, 3 and 10). The run ends `FAILED` with
nothing loaded: `rows_valid` 3, `rows_rejected` 7, `row_issues_total` 7.

**`customers_batch_2024-04-05_fixed.csv`:** the corrected export:
- row 1 (1005, already loaded) is removed
- row 5 (the duplicate 1017) is removed
- `CUST-1020` becomes `1020`
- the email becomes `george.brown@example.com`
- `Active` becomes `active`, and `pending` becomes `inactive`

That leaves 8 rows: `1013, 1014, 1017, 1020, 1018, 1019, 1021, 1022`. The run ends `LOADED`, and the table goes
from 12 to 20 rows.
