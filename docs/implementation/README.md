# Implementation plan and process

Two Claude Code sessions build this app in parallel: a **frontend (FE) session** and a **backend (BE)
session**. This folder keeps them in sync. Read this file before starting any work.

## The goal

A user picks an **existing target table**, uploads **one CSV file** to load into it, and the application runs a
real load pipeline:

```
target table → upload CSV → run pipeline (parse, validate against the table's schema, load)
   → LOADED: the rows are in the table
   → FAILED: nothing is loaded; real failure evidence (checks, row issues, pipeline log)
        → AI investigation of that evidence: root cause, evidence, recommended fix
        → the user corrects the file and uploads it again → validation passes → LOADED
```

The AI only ever sees evidence the application produced for that run. Nothing is hardcoded.

## Phases

| Phase | Goal | Status |
|---|---|---|
| 0. Setup | This folder, the API contract, and the process | Done |
| 1. Two-file upload | Upload a source CSV and a target CSV to compare | **Cancelled.** Replaced by Phase 2. Don't implement `phase-1/` |
| 2. Target load flow | Target tables in SQLite, single-CSV runs, validation from the table schema, all-or-nothing load, AI investigation of a failed run, corrected re-upload | Contract ready |
| 3. Finishing | Demo walkthrough, one-command start, polish, anything left from Phase 2 | Not started |

### Decisions already made
- Target tables live in a **SQLite** database file owned by the backend. They're created and seeded when the
  backend starts, and a reset endpoint restores the seed data so the demo can be repeated.
- The validation rules come from the **target table's schema** (types, nullability, primary key, formats,
  allowed values). They are not hardcoded per demo file.
- **All or nothing:** if any check fails, no rows are loaded.
- A failed run is a normal result, not an HTTP error. Creating a run always returns `201`, with `status`
  `LOADED` or `FAILED`.
- Runs, including their investigation result, are stored **in memory** and lost when the backend restarts.
  The target table data persists in SQLite.
- Upload limit: **10 MB and 50,000 data rows** for the CSV.
- The backend owns the demo files (a failing file and its corrected version) and serves them. The frontend
  doesn't bundle copies.
- The original demo endpoints (`/api/demo/*`, `/api/investigate`) stay unchanged and keep working.
- The code stays in its current layout (`src/`, `tests/`, `frontend/`).

## Who owns what

| Path | Owner | Notes |
|---|---|---|
| `src/`, `tests/`, `data/`, `requirements.txt`, `example_usage.py`, root `.env.example` | BE session | |
| `frontend/` | FE session | |
| `docs/implementation/` | FE session | The BE session only fills in the "Questions" and "Completion report" sections of its own phase file |
| Root `README.md` | Shared | BE edits the backend and run sections; FE edits the frontend sections |
| `CLAUDE.md` | FE session | |

Never edit files the other session owns. If you need a change there, write it under "Questions for the other
session" in your phase file and stop working on that item.

## The API contract

[`CONTRACT.md`](CONTRACT.md) is the single source of truth for every endpoint, request, response and error.
- Only the FE session changes it, and only between phases.
- The BE session implements it exactly: same paths, field names, types, status codes and error codes.
  If something in it is wrong or impossible, don't improvise. Write it under "Questions for the other
  session" in your phase file, implement everything else, and report it in the completion report.
- The FE session builds against the examples in the contract, so both sessions can work at the same time.

## How a phase runs

1. **Start.** The FE session writes `phase-N/BE.md` and `phase-N/FE.md`, updates `CONTRACT.md`, and
   merges these docs into `main`.
2. **Sync.** Both sessions pull the latest `main`, then each creates its own working branch from it.
3. **Build.** Each session follows its own instruction file only, runs the checks listed there, and pushes.
4. **Merge the backend first.** The BE pull request merges into `main` first. The FE session then brings
   `main` into its branch, tests against the real backend, and its pull request merges second.
5. **Close.** The FE session marks the phase Done in the table above, and both sessions pull `main` again
   before the next phase.

## Definition of done (every phase)

- Backend: `python -m pytest -q` passes, and every new endpoint has tests for its success and error cases.
- Frontend: `npm run build` succeeds, and the flow works against the real backend in a browser.
- The existing demo flow still works.
- The phase file's checklist is complete, and its "Completion report" section is filled in, including
  anything not done and why.
