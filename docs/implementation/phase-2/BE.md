# Phase 2: Backend instructions (target load flow)

For the **BE session**. Read [`../README.md`](../README.md) (process and ownership) and the "Phase 2" section
of [`../CONTRACT.md`](../CONTRACT.md) first. The contract is the spec: paths, field names, types, status
codes, error codes, check names, step names and the demo data. This file says how to build it and what to
test. Phase 1 was cancelled; ignore `phase-1/`.

## Goal

A user picks the `customers` target table, uploads one CSV, and the backend runs a real pipeline: parse,
validate against the table's schema, and load into SQLite all-or-nothing. A failed run carries real
evidence, the AI can investigate that run, and a corrected re-upload loads successfully.

## Before you start

1. Pull the latest `main` and create your working branch from it.
2. Confirm the baseline: `pip install -r requirements.txt && python -m pytest -q`. There should be 81 passing tests.
3. Only touch `src/`, `tests/`, `data/`, `requirements.txt`, the root `.env.example`, and the backend and run
   sections of the root `README.md`. Don't touch `frontend/` or `docs/implementation/`, except the
   "Questions" and "Completion report" sections at the bottom of this file.

## Keep working (don't change)

- `GET /health`, `GET /api/demo/scenario`, `GET /api/demo/run` and `POST /api/investigate`, and their responses.
- The investigation service's prompt, parsing, one retry, citation check and trace (`src/investigation/`).
  Extend it for runs as described below; don't fork or rewrite it.
- All 81 existing tests must still pass unchanged.

## Tasks

### 1. Dependency and configuration
- Add `python-multipart>=0.0.9` to `requirements.txt`.
- Add a `TARGET_DB_PATH` env var, defaulting to `data/warehouse.sqlite`, to `src/config.py` and the root
  `.env.example`. `*.sqlite` is already git-ignored.

### 2. Target definitions: `src/targets/`
- `registry.py` defines the targets in code:
  - A `TargetSpec` with `target_id`, `name`, `table_name`, `description`, `columns`, seed rows and samples.
  - A `ColumnSpec` matching the contract's `ColumnSpec`.
  - One target, `customers`, exactly as in the contract, with the same descriptions and rules.
- Generate the `ddl` string from the `TargetSpec`. Don't hand-write it separately, so the DDL can never
  disagree with the checks. Use the formatting shown in the contract.
- Keep the seed rows and the two demo CSVs as files under `data/targets/customers/`: `seed.csv`,
  `customers_batch_2024-04-05.csv` and `customers_batch_2024-04-05_fixed.csv`. Their content must match the
  "demo data" section of the contract exactly, including the `customer_id` order. Use realistic names, emails,
  `country` codes and dates. Every value not named as a defect must pass every check.

### 3. Database: `src/targets/db.py`
- Use `sqlite3` from the standard library. No ORM.
- On app startup (a FastAPI lifespan handler), create each target table if it doesn't exist and seed it if
  it's empty. Existing rows persist across restarts.
- `reset(target_id)`: drop, recreate and re-seed the table in one transaction.
- `count_rows`, `recent_rows(limit=20)` (newest first by `rowid`, with typed values) and `existing_keys(column)`.
- Every path must accept an injected database path, so tests can use `tmp_path`. Expose it through a FastAPI
  dependency that tests can override.

### 4. The pipeline: `src/runs/`
- `ingest.py`: read the upload in chunks with the 10 MB limit.
  - Decode as `utf-8-sig`.
  - Parse with `pd.read_csv(io.StringIO(text), dtype=str, keep_default_na=False, na_values=[""])`, the
    same settings as `src/data/scenario.py::load_scenario`.
  - Detect empty or duplicate header names from the raw header line using the `csv` module, because pandas
    silently renames duplicates.
  - Map each problem to the contract's `413`/`422` codes. None of these create a run.
- `checks.py`: generate the checks from the `TargetSpec`, with the names, order, severities and skip rules in
  the contract's "Validation checks" table.
  - Reuse `src/validation/suite.py`'s `CheckResult` dataclass and its evidence cap (`MAX_EVIDENCE`,
    `"... and N more"`). Don't create a third result shape.
  - Each check returns its `CheckResult` and its `RowIssue`s.
  - Evidence lines name the row, for example `"row 7: email 'george.brown@example' is not a valid email address"`.
  - `key_conflict` reads existing keys from the database. Compare keys as integers for `INTEGER` columns;
    values that don't parse are left to `type.*`.
- `pipeline.py`: run the four steps from the contract in order, timing each one (`duration_ms`).
  - Build `logs` as the steps actually run, in the `"HH:MM:SS LEVEL message"` format.
  - Compute `summary`. `rows_valid` is the number of rows with no issues; `rows_rejected` is the number of
    rows with at least one.
  - `load` converts types (`INTEGER` to `int`, others as text) and inserts every row in one transaction. If
    SQLite raises, roll back, mark `load` `FAILED` with the database's error message, log it as `ERROR`, and
    end the run `FAILED`. With the checks passing this shouldn't happen, but it must be handled.
- `store.py`: an in-memory run store behind a `threading.Lock`, holding up to 100 runs and evicting the oldest.
  - Each record keeps the `Run` response data plus what the investigation needs: the parsed DataFrame's
    profile and the full list of row issues.
  - Run IDs: `"run_" + secrets.token_hex(6)`.
  - `attempt` and `previous_run_id` are as in the contract. A `previous_run_id` that is unknown, or belongs to
    another target, is `422 invalid_previous_run`, and the upload is not processed.

### 5. Investigating a run
- In `src/investigation/service.py`, add `investigate_run(context, provider)`. It builds the evidence package
  for one run and calls the existing `_ask_llm`. Extend the service without breaking current callers:
  - `evidence_catalog` must also emit `target.schema` (for a `target_schema` context key), `dataset.upload`
    (key `upload`) and `dataset.row_issues` (key `row_issues`). The run's pipeline record goes under
    `execution_evidence.pipeline_run`, so the existing code already emits `pipeline.run_log`.
  - `_EVIDENCE_REF` must also accept the `target.` prefix, so citation warnings work for `target.schema`.
  - The context contains:
    - `pipeline_name`: the target name plus `" load"`
    - `pipeline_description`: one sentence built from the `TargetSpec`, stating the table, the
      all-or-nothing load and the file name
    - `target_schema`: `{table_name, ddl, columns}`
    - `execution_evidence`: `{execution_summary: <run summary>, pipeline_run: {status, steps, logs}}`
    - `validation_results`: every `CheckResult` dict
    - `upload`: `{file_name, row_count, columns, null_counts}`
    - `row_issues`: the first 50 issues
  - Never send raw rows beyond those 50 issues. Put no expected conclusion in the prompt or context.
- Map errors to the contract's structured codes: `LLMError` gives `502 llm_error`, `LLMTimeoutError` gives
  `504 llm_timeout`, a missing key gives `503 llm_not_configured`, and anything else gives
  `500 internal_error`. `InvestigationParseError` after the retry is `502 llm_error`.
- Store the returned `InvestigateResponse` on the run, replacing any earlier one.

### 6. API: `src/api/main.py` and `src/api/schemas.py`
- Add Pydantic models for every Phase 2 shape (`TargetSummary`, `TargetDetail`, `ColumnSpec`, `Rule`,
  `SampleFile`, `Run`, `RunSummaryCounts`, `PipelineStep`, `RowIssue`, `RunListItem`, `ErrorDetail`), and the
  8 endpoints from the contract.
- Build every structured error with one helper, so the `{detail: {code, message, field}}` form is identical
  everywhere.
- The LLM provider dependency for `POST /api/runs/{run_id}/investigate` must be overridable exactly like the
  existing `get_llm_provider`, so tests never call a real LLM.

### 7. Tests (new files; keep the existing ones passing)
Use `TestClient`, a `tmp_path` database per test, and the demo files in `data/targets/customers/`.
Mock the LLM only at the provider boundary, with `tests/llm_stub.py`. Cover at least:
- **Targets:**
  - The list and detail match the contract: 12 rows, 6 columns, the rules, `recent_rows` newest first with
    integer IDs, and both samples.
  - The DDL is generated from the spec.
  - Sample download returns `text/csv`; an unknown file is `404 sample_not_found`.
  - An unknown target is `404 target_not_found`.
- **Failing demo file:**
  - `201`, `FAILED`, exactly 15 checks with 5 failed (the 5 named in the contract).
  - `row_issues_total` is 7, with the right `row_number`s and checks.
  - `rows_valid` 3, `rows_rejected` 7, `rows_loaded` 0, and the table still has 12 rows.
  - Steps are `SUCCESS, SUCCESS, FAILED, SKIPPED`.
  - The logs contain an `ERROR` line.
- **Corrected demo file**, with `previous_run_id` set to the failed run:
  - `201`, `LOADED`, `attempt` 2, 15 checks all passed, 8 rows loaded, and the table goes from 12 to 20.
  - `recent_rows` shows the new rows.
- Run history is newest first. `GET /api/runs/{id}` round-trips. An unknown run is `404 run_not_found`.
- Uploading the corrected file twice: the second upload fails `key_conflict.customer_id` for all 8 rows.
- Reset restores 12 rows.
- **Each upload error code:**
  - `file_too_large` (monkeypatch the limit)
  - `invalid_encoding`
  - `invalid_csv` for an empty file, a header with no rows, and a duplicate or empty header
  - `too_many_rows` (monkeypatch the limit)
  - `invalid_previous_run` for an unknown ID and for another target's run
  - None of these create a run.
- **Missing and extra columns:** `schema.missing_columns` fails; the checks for missing columns show
  `"Not run: column missing from file"`; `schema.unexpected_columns` fails for an extra column; still exactly
  15 checks.
- **`load` failure:** monkeypatch the insert to raise `sqlite3.IntegrityError`. The run is `FAILED`, `load` is
  `FAILED`, and the table is unchanged.
- **Investigation:**
  - A failed run gives `200` in the `InvestigateResponse` shape.
  - The prompt context contains `target.schema`, `pipeline.run_log`, `validation.*`, `dataset.upload` and
    `dataset.row_issues` in `available_evidence`, and contains no more than 50 row issues.
  - The result is stored and returned by `GET /api/runs/{id}`.
  - A `LOADED` run is `409 run_not_failed`.
  - A missing key, a timeout, a provider error and invalid output after the retry map to `503`, `504`, `502`
    and `502`.

### 8. Docs
- Root `README.md`: add `src/targets/` and `src/runs/` to the backend components table, and one short
  paragraph describing the target load flow and the SQLite file.

## Done checklist

- [ ] `python -m pytest -q` passes: the 81 existing tests plus the new ones
- [ ] Manual check with the backend running:
  - [ ] `curl -F file=@data/targets/customers/customers_batch_2024-04-05.csv localhost:8000/api/targets/customers/runs`
        returns `201` `FAILED` with the contract's numbers
  - [ ] The fixed file with `-F previous_run_id=<id>` returns `LOADED`, and `GET /api/targets/customers`
        shows 20 rows
  - [ ] `POST /api/targets/customers/reset` returns the table to 12 rows
- [ ] With a real `LLM_API_KEY`, `POST /api/runs/<failed id>/investigate` returns a result that cites the run's
      evidence IDs
- [ ] The existing endpoints and tests are unchanged
- [ ] No files outside your ownership were changed
- [ ] The completion report below is filled in, and the work is pushed with a pull request into `main`

## Out of scope for Phase 2

More than one target table, user-defined targets, editing data in the app, applying the AI's fix
automatically, executing the generated regression test, authentication, and background jobs (the pipeline
runs synchronously).

---

## Questions for the other session

_(BE session: add anything in the contract that is unclear or impossible, then continue with the rest.)_

## Completion report

_(BE session: fill in when done.)_
- Branch / PR:
- Test count before → after:
- Deviations from the contract (should be none):
- Anything not done, and why:
