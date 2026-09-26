# Phase 2: Frontend instructions (target load flow)

For the **FE session**. Read [`../README.md`](../README.md) and the "Phase 2" section of
[`../CONTRACT.md`](../CONTRACT.md) first. Phase 1 was cancelled; its UI (commit `d5a5261`, never merged) is
not used, apart from pieces reused below.

## Goal

One guided flow: **pick a target table → upload a CSV → see the run → on failure, see the evidence and
investigate with AI → upload a corrected file → see it load.**

## Before you start

1. Pull the latest `main` and create your working branch from it.
2. Confirm the baseline: `cd frontend && npm ci && npm run build`.
3. Only touch `frontend/`, `docs/implementation/`, `CLAUDE.md`, and the frontend sections of the root `README.md`.

## Screens

Keep simple view state in `App.jsx`, with no router. Clicking the app title returns to the target list.

### 1. Target list (the start screen)
- One card per target from `GET /api/targets`: name, description, table name, row count and column count.
  Clicking a card opens the target.
- A secondary link, "Explore the original case study", opens the existing demo page (`InvestigationPage`)
  unchanged.
- In sample-data mode (no `VITE_API_BASE_URL`), explain that the load flow needs the backend, and keep only
  the case-study link.

### 2. Target page (`GET /api/targets/{id}`)
- Header: name, table name, description, current row count, and a **Reset table** button. The reset asks for
  confirmation, calls `POST .../reset`, and refreshes the page.
- **Schema** tab: a columns table (name, type, nullable, primary key, rules shown as readable text) and the
  DDL in the existing `CodeBlock`.
- **Rows** tab: `recent_rows` in a table, newest first.
- **History** tab: `GET .../runs` as a list with status badges, attempts linked by `previous_run_id`, and file
  names. Clicking one opens that run.
- **Upload** panel, prominent:
  - One CSV drop area. Reuse `FileDrop` from `d5a5261`, with the 10 MB check in the browser.
  - A **Run pipeline** button.
  - The demo files from `samples`, each with a download link and its expected outcome shown as a badge.
  - Upload errors (`413`/`422`) appear under the drop area using `detail.message`.

### 3. Run page (`POST .../runs` result, or `GET /api/runs/{id}`)
- **Header:**
  - A large status: **Loaded** (green) or **Failed – nothing was loaded** (red).
  - The file name, run ID, attempt number, and a link to the previous run if there is one.
  - The summary numbers: received, valid, rejected, loaded, and table rows before → after.
- **Pipeline:** the four steps as a timeline with status, rows in → out, duration and message. Reuse
  `PipelineSteps` or adapt it; it currently expects the demo's step fields. The logs go in the existing
  `LogViewer`.
- **Checks:** reuse `ValidationChecks`. The `CheckResult` shape is the same as the demo's.
- **Row issues:** a table of `row_number`, column, value, check and message, filterable by check, with a
  "showing N of `row_issues_total`" note.
- **When `FAILED`:**
  - The AI investigation panel. Reuse `InvestigationPanel`, `HypothesisList`, `CitedList` and `CodeBlock`,
    because the response shape is the same `InvestigateResponse`.
  - It calls `POST /api/runs/{id}/investigate`. A stored `investigation` shows at once, with **Re-run**
    available.
  - Error codes map to clear messages: `llm_not_configured` and the others.
  - **Upload a corrected file:** the same drop area, sending `previous_run_id` = this run.
- **When `LOADED`:**
  - A success state: "8 rows loaded into customers (12 → 20)".
  - If the run corrects an earlier failure, show the chain, for example "Attempt 2 fixed attempt 1".
  - **View the table** returns to the target page's Rows tab.
- A reload keeps the current run: save the run ID in `sessionStorage` (guarded with try/catch), and restore it
  with `GET /api/runs/{id}`. On `run_not_found`, return to the target page with the message.

## API client (`src/services/api.js`)

- Bring back `ApiError` and structured error parsing from `d5a5261`: `status`, `code`, `field` and `message`
  for all three error forms.
- Add `listTargets`, `getTarget`, `resetTarget`, `sampleUrl(targetId, fileName)`,
  `createRun(targetId, file, previousRunId?)`, `listRuns`, `getRun` and `investigateRun`. Keep the existing
  demo calls unchanged.
- Add JSDoc types for every Phase 2 shape in `src/types/index.js`.
- Don't hardcode check names, column names or row counts anywhere in the load flow. Render what the API
  returns. The demo page keeps its existing assumptions.

## Testing

- **Before the backend is merged:** test against a local stub that returns the contract's examples and follows
  its rules: the failing file gives `FAILED` with the listed issues, the fixed file gives `LOADED`, reset
  works, and the errors are as listed. Don't commit the stub.
- **After the backend is merged:** bring `main` into this branch. In a browser against the real backend:
  1. Pick Customers; see 12 rows, the schema and the DDL.
  2. Download and upload the failing demo file. See `FAILED`, 5 failed checks, 7 row issues, `load` skipped,
     and still 12 rows.
  3. Investigate. With a real key, check that the result cites this run's evidence IDs. Without one, check
     the `llm_not_configured` message.
  4. Upload the fixed file as a correction. See `LOADED`, attempt 2 linked to attempt 1, and 12 → 20 rows.
  5. Reload on a run page, then restart the backend and reload again.
  6. Reset the table.
  7. Open the case-study demo; it's unchanged.
  8. Check a phone-width layout (375 px) and dark mode.

## Done checklist

- [ ] `npm run build` succeeds
- [ ] Every browser test above passes against the real backend
- [ ] No check, column or count is hardcoded in the load flow
- [ ] The completion report below is filled in; the PR merges after the backend PR

---

## Questions for the other session

## Completion report
- Branch / PR:
- Deviations from the contract:
- Anything not done, and why:
